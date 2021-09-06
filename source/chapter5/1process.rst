与进程有关的重要系统调用
================================================

重要系统调用
------------------------------------------------------------

fork 系统调用
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. code-block:: rust

    /// 功能：由当前进程 fork 出一个子进程。
    /// 返回值：对于子进程返回 0，对于当前进程则返回子进程的 PID 。
    /// syscall ID：220
    pub fn sys_fork() -> isize;

exec 系统调用
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. code-block:: rust

    /// 功能：将当前进程的地址空间清空并加载一个特定的可执行文件，返回用户态后开始它的执行。
    /// 参数：path 给出了要加载的可执行文件的名字；
    /// 返回值：如果出错的话（如找不到名字相符的可执行文件）则返回 -1，否则不应该返回。
    /// syscall ID：221
    pub fn sys_exec(path: &str) -> isize;

在实际进行系统调用时，我们只会将起始地址传给内核（对标 C 语言仅会传入一个 ``char*`` ）。
需要应用负责在传入的字符串的末尾加上一个 ``\0`` ，这样内核才能知道字符串的长度。

利用 ``fork`` 和 ``exec`` 的组合，我们很容易在一个进程内 ``fork`` 出一个子进程并执行一个特定的可执行文件。

waitpid 系统调用
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. code-block:: rust

    /// 功能：当前进程等待一个子进程变为僵尸进程，回收其全部资源并收集其返回值。
    /// 参数：pid 表示要等待的子进程的进程 ID，如果为 -1 的话表示等待任意一个子进程；
    /// exit_code 表示保存子进程返回值的地址，如果这个地址为 0 的话表示不必保存。
    /// 返回值：如果要等待的子进程不存在则返回 -1；否则如果要等待的子进程均未结束则返回 -2；
    /// 否则返回结束的子进程的进程 ID。
    /// syscall ID：260
    pub fn sys_waitpid(pid: isize, exit_code: *mut i32) -> isize;


``sys_waitpid`` 在用户库中被封装成两个不同的 API， ``wait(exit_code: &mut i32)`` 和 ``waitpid(pid: usize, exit_code: &mut i32)``，
前者用于等待任意一个子进程，后者用于等待特定子进程。实现的策略是，如果子进程还未结束，就以 yield 让出时间片：

.. code-block:: rust
    :linenos:

    // user/src/lib.rs

    pub fn wait(exit_code: &mut i32) -> isize {
        loop {
            match sys_waitpid(-1, exit_code as *mut _) {
                -2 => { sys_yield(); }
                n => { return n; }
            }
        }
    }


应用程序示例
-----------------------------------------------

借助这三个重要系统调用，我们可以开发功能更强大的应用。下面是两个案例： **用户初始程序-init** 和 **shell程序-user_shell** 。

用户初始程序-initproc
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

在内核初始化完毕之后会创建一个进程，即 **用户初始进程** (Initial Process) ，
它是目前在内核中以硬编码方式创建的唯一一个进程。其他所有的进程都从它开始通过 ``fork+exec`` 的系统调用创建。

.. code-block:: rust
    :linenos:

    // user/src/bin/initproc.rs

    #![no_std]
    #![no_main]

    #[macro_use]
    extern crate user_lib;

    use user_lib::{
        fork,
        wait,
        exec,
        yield_,
    };

    #[no_mangle]
    fn main() -> i32 {
        if fork() == 0 {
            exec("user_shell\0");
        } else {
            loop {
                let mut exit_code: i32 = 0;
                let pid = wait(&mut exit_code);
                if pid == -1 {
                    yield_();
                    continue;
                }
                println!(
                    "[initproc] Released a zombie process, pid={}, exit_code={}",
                    pid,
                    exit_code,
                );
            }
        }
        0
    }

- 第 19 行为 ``fork`` 出的子进程分支，通过 ``exec`` 启动shell程序 ``user_shell`` ，
  注意我们需要在字符串末尾手动加入 ``\0`` 。
- 第 21 行开始则为父进程分支，表示用户初始程序-initproc自身。它不断循环调用 ``wait`` 来等待并回收系统中的僵尸进程占据的资源。
  如果回收成功的话则会打印一条报告信息给出被回收子进程的 PID 和返回值；否则就 ``yield_`` 交出 CPU 资源并在下次轮到它执行的时候再回收看看。
  

shell程序-user_shell
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

user_shell 需要捕获我们的输入并进行解析处理，我们需要加入一个新的用于输入的系统调用：

.. code-block:: rust

    /// 功能：从文件中读取一段内容到缓冲区。
    /// 参数：fd 是待读取文件的文件描述符，切片 buffer 则给出缓冲区。
    /// 返回值：如果出现了错误则返回 -1，否则返回实际读到的字节数。
    /// syscall ID：63
    pub fn sys_read(fd: usize, buffer: &mut [u8]) -> isize;

实际调用时，我们必须要同时向内核提供缓冲区的起始地址及长度：

.. code-block:: rust

    // user/src/syscall.rs

    pub fn sys_read(fd: usize, buffer: &mut [u8]) -> isize {
        syscall(SYSCALL_READ, [fd, buffer.as_mut_ptr() as usize, buffer.len()])
    }

我们在用户库中将其进一步封装成每次能够从 **标准输入** 中获取一个字符的 ``getchar`` 函数：

shell程序- ``user_shell`` 实现如下：

.. code-block:: rust
    :linenos:
    :emphasize-lines: 28,53,61

    // user/src/bin/user_shell.rs

    #![no_std]
    #![no_main]

    extern crate alloc;

    #[macro_use]
    extern crate user_lib;

    const LF: u8 = 0x0au8;
    const CR: u8 = 0x0du8;
    const DL: u8 = 0x7fu8;
    const BS: u8 = 0x08u8;

    use alloc::string::String;
    use user_lib::{fork, exec, waitpid, yield_};
    use user_lib::console::getchar;

    #[no_mangle]
    pub fn main() -> i32 {
        println!("Rust user shell");
        let mut line: String = String::new();
        print!(">> ");
        loop {
            let c = getchar();
            match c {
                LF | CR => {
                    println!("");
                    if !line.is_empty() {
                        line.push('\0');
                        let pid = fork();
                        if pid == 0 {
                            // child process
                            if exec(line.as_str()) == -1 {
                                println!("Error when executing!");
                                return -4;
                            }
                            unreachable!();
                        } else {
                            let mut exit_code: i32 = 0;
                            let exit_pid = waitpid(pid as usize, &mut exit_code);
                            assert_eq!(pid, exit_pid);
                            println!(
                                "Shell: Process {} exited with code {}",
                                pid, exit_code
                            );
                        }
                        line.clear();
                    }
                    print!(">> ");
                }
                BS | DL => {
                    if !line.is_empty() {
                        print!("{}", BS as char);
                        print!(" ");
                        print!("{}", BS as char);
                        line.pop();
                    }
                }
                _ => {
                    print!("{}", c as char);
                    line.push(c as char);
                }
            }
        }
    }

可以看到，在以第 25 行开头的主循环中，每次都是调用 ``getchar`` 获取一个用户输入的字符，并根据它相应进行一些动作。第 23 行声明的字符串 ``line`` 则维护着用户当前输入的命令内容，它也在不断发生变化。

- 如果用户输入回车键（第 28 行），那么user_shell 会 fork 出一个子进程（第 34 行开始）并试图通过 ``exec`` 系统调用执行一个应用，应用的名字在字符串 ``line`` 中给出。这里我们需要注意的是由于子进程是从user_shell 进程中 fork 出来的，它们除了 fork 的返回值不同之外均相同，自然也可以看到一个和user_shell 进程维护的版本相同的字符串 ``line`` 。第 35 行对 ``exec`` 的返回值进行了判断，如果返回值为 -1 的话目前说明在应用管理器中找不到名字相同的应用，此时子进程就直接打印错误信息并退出；反之 ``exec`` 则根本不会返回，而是开始执行目标应用。

  fork 之后的 user_shell 进程自己的逻辑可以在第 41 行找到。可以看出它只是在等待 fork 出来的子进程结束并回收掉它的资源，还会顺带收集子进程的退出状态并打印出来。
- 如果用户输入退格键（第 53 行），首先我们需要将屏幕上当前行的最后一个字符用空格替换掉，这可以通过输入一个特殊的退格字节 ``BS`` 来实现。其次，user_shell 进程内维护的 ``line`` 也需要弹出最后一个字符。
- 如果用户输入了一个其他字符（第 61 行），它将会被视为用户的正常输入，我们直接将它打印在屏幕上并加入到 ``line`` 中。
