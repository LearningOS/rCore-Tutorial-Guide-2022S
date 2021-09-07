文件系统接口
=================================================

简易文件与目录抽象
-------------------------------------------------

与课堂所学相比，我们实现的文件系统进行了很大的简化：

- 扁平化：仅存在根目录 ``/`` 一个目录，所有的文件都放在根目录内。直接以文件名索引文件。
- 不设置用户和用户组概念，不记录文件访问/修改的任何时间戳，不支持软硬链接。
- 只实现了最基本的文件系统相关系统调用。

打开与读写文件的系统调用
--------------------------------------------------

文件打开
++++++++++++++++++++++++++++++++++++++++++++++++++

.. code-block:: rust

    /// 功能：打开一个常规文件，并返回可以访问它的文件描述符。
    /// 参数：path 描述要打开的文件的文件名（简单起见，文件系统不需要支持目录，所有的文件都放在根目录 / 下），
    /// flags 描述打开文件的标志，具体含义下面给出。
    /// 返回值：如果出现了错误则返回 -1，否则返回打开常规文件的文件描述符。可能的错误原因是：文件不存在。
    /// syscall ID：56
    pub fn sys_open(path: *const u8, flags: u32) -> isize;

目前我们的内核支持以下几种标志（多种不同标志可能共存）：

- 如果 ``flags`` 为 0，则表示以只读模式 *RDONLY* 打开；
- 如果 ``flags`` 第 0 位被设置（0x001），表示以只写模式 *WRONLY* 打开；
- 如果 ``flags`` 第 1 位被设置（0x002），表示既可读又可写 *RDWR* ；
- 如果 ``flags`` 第 9 位被设置（0x200），表示允许创建文件 *CREATE* ，在找不到该文件的时候应创建文件；如果该文件已经存在则应该将该文件的大小归零；
- 如果 ``flags`` 第 10 位被设置（0x400），则在打开文件的时候应该清空文件的内容并将该文件的大小归零，也即 *TRUNC* 。

注意 ``flags`` 里面的权限设置只能控制进程对本次打开的文件的访问。一般情况下，在打开文件的时候首先需要经过文件系统的权限检查，比如一个文件自身不允许写入，那么进程自然也就不能以 *WRONLY* 或 *RDWR* 标志打开文件。但在我们简化版的文件系统中文件不进行权限设置，这一步就可以绕过。

在用户库 ``user_lib`` 中，我们将该系统调用封装为 ``open`` 接口：

.. code-block:: rust

    // user/src/lib.rs

    bitflags! {
        pub struct OpenFlags: u32 {
            const RDONLY = 0;
            const WRONLY = 1 << 0;
            const RDWR = 1 << 1;
            const CREATE = 1 << 9;
            const TRUNC = 1 << 10;
        }
    }

    pub fn open(path: &str, flags: OpenFlags) -> isize {
        sys_open(path, flags.bits) 
    }

借助 ``bitflags!`` 宏我们将一个 ``u32`` 的 flags 包装为一个 ``OpenFlags`` 结构体更易使用，它的 ``bits`` 字段可以将自身转回 ``u32`` ，它也会被传给 ``sys_open`` ：

.. code-block:: rust

    // user/src/syscall.rs

    const SYSCALL_OPEN: usize = 56;

    pub fn sys_open(path: &str, flags: u32) -> isize {
        syscall(SYSCALL_OPEN, [path.as_ptr() as usize, flags as usize, 0])
    }

我们在 ``sys_open`` 传给内核的两个参数只有待打开文件的文件名字符串的起始地址（和之前一样，我们需要保证该字符串以 ``\0`` 结尾）还有标志位。由于每个通用寄存器为 64 位，我们需要先将 ``u32`` 的 ``flags`` 转换为 ``usize`` 。

文件的顺序读写
++++++++++++++++++++++++++++++++++++++++++++++++++

在打开一个文件之后，我们就可以用之前的 ``sys_read/sys_write`` 两个系统调用来对它进行读写了。需要注意的是，常规文件的读写模式和之前介绍过的几种文件有所不同。标准输入输出和匿名管道都属于一种流式读写，而常规文件则是顺序读写和随机读写的结合。由于常规文件可以看成一段字节序列，我们应该能够随意读写它的任一段区间的数据，即随机读写。然而用户仅仅通过 ``sys_read/sys_write`` 两个系统调用不能做到这一点。

事实上，进程为每个它打开的常规文件维护了一个偏移量，在刚打开时初始值一般为 0 字节。当 ``sys_read/sys_write`` 的时候，将会从文件字节序列偏移量的位置开始 **顺序** 把数据读到应用缓冲区/从应用缓冲区写入数据。操作完成之后，偏移量向后移动读取/写入的实际字节数。这意味着，下次 ``sys_read/sys_write`` 将会从刚刚读取/写入之后的位置继续。如果仅使用 ``sys_read/sys_write`` 的话，则只能从头到尾顺序对文件进行读写。当我们需要从头开始重新写入或读取的话，只能通过 ``sys_close`` 关闭并重新打开文件来将偏移量重置为 0。为了解决这种问题，有另一个系统调用 ``sys_lseek`` 可以调整进程打开的一个常规文件的偏移量，这样便能对文件进行随机读写。在本教程中并未实现这个系统调用，因为顺序文件读写就已经足够了。顺带一提，在文件系统的底层实现中都是对文件进行随机读写的。
 
.. _filetest-simple:

下面我们从本章的测试用例 ``filetest_simple`` 来介绍文件系统接口的使用方法：

.. code-block:: rust
    :linenos:

    // user/src/bin/filetest_simple.rs

    #![no_std]
    #![no_main]

    #[macro_use]
    extern crate user_lib;

    use user_lib::{
        open,
        close,
        read,
        write,
        OpenFlags,
    };

    #[no_mangle]
    pub fn main() -> i32 {
        let test_str = "Hello, world!";
        let filea = "filea\0";
        let fd = open(filea, OpenFlags::CREATE | OpenFlags::WRONLY);
        assert!(fd > 0);
        let fd = fd as usize;
        write(fd, test_str.as_bytes());
        close(fd);

        let fd = open(filea, OpenFlags::RDONLY);
        assert!(fd > 0);
        let fd = fd as usize;
        let mut buffer = [0u8; 100];
        let read_len = read(fd, &mut buffer) as usize;
        close(fd);

        assert_eq!(
            test_str,
            core::str::from_utf8(&buffer[..read_len]).unwrap(),
        );
        println!("file_test passed!");
        0
    }

- 第 20~25 行，我们打开文件 ``filea`` ，向其中写入字符串 ``Hello, world!`` 而后关闭文件。这里需要注意的是我们需要为字符串字面量手动加上 ``\0`` 作为结尾。在打开文件时 *CREATE* 标志使得如果 ``filea`` 原本不存在，文件系统会自动创建一个同名文件，如果已经存在的话则会清空它的内容。而 *WRONLY* 使得此次只能写入该文件而不能读取。
- 第 27~32 行，我们以只读 *RDONLY* 的方式将文件 ``filea`` 的内容读取到缓冲区 ``buffer`` 中。注意我们很清楚 ``filea`` 的总大小不超过缓冲区的大小，因此通过单次 ``read`` 即可将 ``filea`` 的内容全部读取出来。而更常见的情况是需要进行多次 ``read`` 直到它的返回值为 0 才能确认文件的内容已被读取完毕了。
- 最后的第 34~38 行我们确认从 ``filea`` 读取到的内容和之前写入的一致，则测试通过。
