.. _term-print-kernelminienv:

构建裸机执行环境
=================================

.. toctree::
   :hidden:
   :maxdepth: 5

本节导读
-------------------------------

有了上一节实现的用户态的最小执行环境，稍加改造，就可以完成裸机上的最小执行环境了。
我们将把 ``Hello world!`` 应用程序从用户态态搬到内核态。 


裸机启动过程
----------------------------

用 QEMU 软件 ``qemu-system-riscv64`` 来模拟 RISC-V 64 计算机。加载内核程序的命令如下：

.. code-block:: bash
    
    qemu-system-riscv64 \
		-machine virt \
		-nographic \
		-bios $(BOOTLOADER) \
		-device loader,file=$(KERNEL_BIN),addr=$(KERNEL_ENTRY_PA)


-  ``-bios $(BOOTLOADER)`` 意味着硬件加载了一个 BootLoader 程序 -- RustSBI
-  ``-device loader,file=$(KERNEL_BIN),addr=$(KERNEL_ENTRY_PA)`` 表示硬件内存中的特定位置 ``$(KERNEL_ENTRY_PA)`` 放置了操作系统的二进制代码 ``$(KERNEL_BIN)`` 。 ``$(KERNEL_ENTRY_PA)`` 的值是 ``0x80200000`` 。

当我们执行包含上次启动参数的qemu-system-riscv64软件，就意味给这台虚拟的RISC-V64计算机加电了。此时，CPU的其它通用寄存器清零，
而PC寄存器会指向 ``0x1000`` 的位置，这里有固化在硬件中的一小段引导代码，它会很快跳转到 ``0x80000000`` RustSBI 处。
RustSBI完成硬件初始化后，
会跳转操作系统的二进制代码 ``$(KERNEL_BIN)`` 所在内存位置 ``0x80200000`` ，执行操作系统的第一条指令。
这时我们的编写的操作系统才开始正式工作。

.. figure:: chap1-intro.png
   :align: center

.. note::

  **操作系统与SBI之间是啥关系？**

  SBI 是 RISC-V 的一种底层规范，RustSBI 是它的一种实现。
  操作系统内核与 RustSBI 的关系有点像应用与操作系统内核的关系，后者向前者提供一定的服务。只是SBI提供的服务很少，
  比如关机，显示字符串等。

实现关机功能
----------------------------

在上一节完成的没有显示功能的用户态最小化执行环境基础上，修改后的代码如下：

.. _term-llvm-sbicall:

.. code-block:: rust

    // bootloader/rustsbi-qemu.bin 直接添加的SBI规范实现的二进制代码，给操作系统提供基本支持服务

    // os/src/sbi.rs
    fn sbi_call(which: usize, arg0: usize, arg1: usize, arg2: usize) -> usize {
    ...

    const SBI_SHUTDOWN: usize = 8;

    pub fn shutdown() -> ! {
        sbi_call(SBI_SHUTDOWN, 0, 0, 0);
        panic!("It should shutdown!");
    }

    // os/src/main.rs
    #[no_mangle]
    extern "C" fn _start() {
        shutdown();
    }


应用程序访问操作系统提供的系统调用的指令是 ``ecall`` ，操作系统访问
RustSBI提供的SBI调用的指令也是 ``ecall`` ，虽然指令一样，但它们所在的特权级是不一样的。简单地说，应用程序位于最弱的用户特权级（User Mode），操作系统位于
很强大的内核特权级（Supervisor Mode），RustSBI位于完全掌控机器的机器特权级（Machine Mode）。下一章会进一步阐释具体细节。

下面是编译执行，结果如下：


.. code-block:: console

  # 编译生成ELF格式的执行文件
  $ cargo build --release
   Compiling os v0.1.0 (/media/chyyuu/ca8c7ba6-51b7-41fc-8430-e29e31e5328f/thecode/rust/os_kernel_lab/os)
    Finished release [optimized] target(s) in 0.15s
  # 把ELF执行文件转成bianary文件
  $ rust-objcopy --binary-architecture=riscv64 target/riscv64gc-unknown-none-elf/release/os --strip-all -O binary target/riscv64gc-unknown-none-elf/release/os.bin

  #加载运行
  $ qemu-system-riscv64 -machine virt -nographic -bios ../bootloader/rustsbi-qemu.bin -device loader,file=target/riscv64gc-unknown-none-elf/release/os.bin,addr=0x80200000
  # 无法退出，风扇狂转，感觉碰到死循环

这样的结果是我们不期望的。问题在哪？仔细查看和思考，操作系统的入口地址不对！对 ``os`` ELF执行程序，通过rust-readobj分析，看到的入口地址不是
RustSBIS约定的 ``0x80200000`` 。我们需要修改 ``os`` ELF执行程序的内存布局。


设置正确的程序内存布局
----------------------------

我们可以通过 **链接脚本** (Linker Script) 调整链接器的行为，使得最终生成的可执行文件的内存布局符合我们的预期。

修改 Cargo 的配置文件来使用我们自己的链接脚本 ``os/src/linker.ld``：

.. code-block::
    :linenos:
    :emphasize-lines: 5,6,7,8

    // os/.cargo/config
    [build]
    target = "riscv64gc-unknown-none-elf"

    [target.riscv64gc-unknown-none-elf]
    rustflags = [
        "-Clink-arg=-Tsrc/linker.ld", "-Cforce-frame-pointers=yes"
    ]

具体的链接脚本 ``os/src/linker.ld`` 如下：

.. code-block::
    :linenos:

    OUTPUT_ARCH(riscv)
    ENTRY(_start)
    BASE_ADDRESS = 0x80200000;

    SECTIONS
    {
        . = BASE_ADDRESS;
        skernel = .;

        stext = .;
        .text : {
            *(.text.entry)
            *(.text .text.*)
        }

        . = ALIGN(4K);
        etext = .;
        srodata = .;
        .rodata : {
            *(.rodata .rodata.*)
        }

        . = ALIGN(4K);
        erodata = .;
        sdata = .;
        .data : {
            *(.data .data.*)
        }

        . = ALIGN(4K);
        edata = .;
        .bss : {
            *(.bss.stack)
            sbss = .;
            *(.bss .bss.*)
        }

        . = ALIGN(4K);
        ebss = .;
        ekernel = .;

        /DISCARD/ : {
            *(.eh_frame)
        }
    }

第 1 行我们设置了目标平台为 riscv ；第 2 行我们设置了整个程序的入口点为之前定义的全局符号 ``_start``；
第 3 行定义了一个常量 ``BASE_ADDRESS`` 为 ``0x80200000`` ，也就是我们之前提到的，期望我们自己实现的初始化代码被放在的地址；

从第 5 行开始体现了链接过程中对输入的目标文件的段的合并。其中 ``.`` 表示当前地址。我们还能够看到这样的格式：

.. code-block::

    .rodata : {
        *(.rodata)
    }

冒号前面表示最终生成的可执行文件的一个段的名字，花括号内按照放置顺序，描述将所有输入目标文件的哪些段放在这个段中，每一行格式为 
``<ObjectFile>(SectionName)``，表示目标文件 ``ObjectFile`` 的名为 ``SectionName`` 的段需要被放进去。我们也可以
使用通配符来书写 ``<ObjectFile>`` 和 ``<SectionName>`` 分别表示可能的输入目标文件和段名。因此，最终的合并结果是，在最终可执行文件
中各个常见的段 ``.text, .rodata .data, .bss`` 从低地址到高地址按顺序放置，每个段里面都包括了所有输入目标文件的同名段，
且每个段都有两个全局符号给出了它的开始和结束地址（比如 ``.text`` 段的开始和结束地址分别是 ``stext`` 和 ``etext`` ）。


这样一来，我们就将运行时重建完毕了。在 ``os`` 目录下 ``cargo build --release`` 或者直接 ``make build`` 就能够看到
最终生成的可执行文件 ``target/riscv64gc-unknown-none-elf/release/os`` 。
通过分析，我们看到 ``0x80200000`` 处的代码是我们预期的 ``_start()`` 函数的内容。我们采用刚才的编译运行方式进行试验，发现还是同样的错误结果。
问题出在哪里？我们没有设置好 **栈 stack** ！ 好吧，我们需要考虑如何合理设置 **栈 stack** 。


正确配置栈空间布局
----------------------------

我们自己编写运行时初始化的代码：

.. code-block:: asm
    :linenos:

    # os/src/entry.asm
        .section .text.entry
        .globl _start
    _start:
        la sp, boot_stack_top
        call rust_main

        .section .bss.stack
        .globl boot_stack
    boot_stack:
        .space 4096 * 16
        .globl boot_stack_top
    boot_stack_top:

在这段汇编代码中，我们从第 8 行开始预留了一块大小为 4096 * 16 字节也就是 :math:`64\text{KiB}` 的空间用作接下来要运行的程序的栈空间，
这块栈空间的栈顶地址被全局符号 ``boot_stack_top`` 标识，栈底则被全局符号 ``boot_stack`` 标识。同时，这块栈空间单独作为一个名为 
``.bss.stack`` 的段，之后我们会通过链接脚本来安排它的位置。

从第 2 行开始，我们通过汇编代码实现执行环境的初始化，它其实只有两条指令：第一条指令将 sp 设置为我们预留的栈空间的栈顶位置，于是之后在函数
调用的时候，栈就可以从这里开始向低地址增长了。简单起见，我们目前暂时不考虑 sp 越过了栈底 ``boot_stack`` ，也就是栈溢出的情形，虽然这有
可能导致严重的错误。第二条指令则是通过伪指令 ``call`` 函数调用 ``rust_main`` ，这里的 ``rust_main`` 是一个我们稍后自己编写的应用
入口。因此初始化任务非常简单：正如上面所说的一样，只需要设置栈指针 sp，随后跳转到应用入口即可。这两条指令单独作为一个名为 
``.text.entry`` 的段，且全局符号 ``_start`` 给出了段内第一条指令的地址。

接着，我们在 ``main.rs`` 中嵌入这些汇编代码并声明应用入口 ``rust_main`` ：

.. code-block:: rust
    :linenos:
    :emphasize-lines: 4,8,10,11,12,13

    // os/src/main.rs
    #![no_std]
    #![no_main]
    #![feature(global_asm)]

    mod lang_items;

    global_asm!(include_str!("entry.asm"));

    #[no_mangle]
    pub fn rust_main() -> ! {
        loop {}
    }

背景高亮指出了 ``main.rs`` 中新增的代码。

第 4 行中，我们手动设置 ``global_asm`` 特性来支持在 Rust 代码中嵌入全局汇编代码。第 8 行，我们首先通过 
``include_str!`` 宏将同目录下的汇编代码 ``entry.asm`` 转化为字符串并通过 ``global_asm!`` 宏嵌入到代码中。

从第 10 行开始，
我们声明了应用的入口点 ``rust_main`` ，这里需要注意的是需要通过宏将 ``rust_main`` 标记为 ``#[no_mangle]`` 以避免编译器对它的
名字进行混淆，不然的话在链接的时候， ``entry.asm`` 将找不到 ``main.rs`` 提供的外部符号 ``rust_main`` 从而导致链接失败。

再次使用上节中的编译，生成和运行操作，我们看到QEMU模拟的RISC-V 64计算机 **优雅** 地退出了！

.. code-block:: console

    $ qemu-system-riscv64 \
    > -machine virt \
    > -nographic \
    > -bios ../bootloader/rustsbi-qemu.bin \
    > -device loader,file=target/riscv64gc-unknown-none-elf/release/os.bin,addr=0x80200000 
    [rustsbi] Version 0.1.0
    .______       __    __      _______.___________.  _______..______   __
    |   _  \     |  |  |  |    /       |           | /       ||   _  \ |  |
    |  |_)  |    |  |  |  |   |   (----`---|  |----`|   (----`|  |_)  ||  |
    |      /     |  |  |  |    \   \       |  |      \   \    |   _  < |  |
    |  |\  \----.|  `--'  |.----)   |      |  |  .----)   |   |  |_)  ||  |
    | _| `._____| \______/ |_______/       |__|  |_______/    |______/ |__|

    [rustsbi] Platform: QEMU
    [rustsbi] misa: RV64ACDFIMSU
    [rustsbi] mideleg: 0x222
    [rustsbi] medeleg: 0xb1ab
    [rustsbi] Kernel entry: 0x80200000


清空 .bss 段
----------------------------------

与内存相关的部分太容易出错了， **清零 .bss段** 的工作我们还没有完成。

.. note::

    由于一般应用程序的 ``.bss`` 段在程序正式开始运行之前会被执环境（系统库或操作系统内核）固定初始化为零，
    因此在 ELF 文件中，为了节省磁盘空间，只会记录 ``.bss`` 段的位置，且应用程序的假定在它执行前，其 ``.bss段`` 的数据内容都已是 ``全0`` 。
    如果这块区域不是全零，且执行环境也没提前清零，那么会与应用的假定矛盾，导致程序出错。对于在裸机上执行的应用程序，其执行环境（就是QEMU模拟硬件+“三叶虫”操作系统内核）要负责将 ``.bss`` 所分配到的内存区域全部清零。

落实到我们正在实现的“三叶虫”操作系统内核，我们需要提供清零的 ``clear_bss()`` 函数。此函数属于执行环境，并在执行环境调用
应用程序的 ``rust_main`` 主函数前，把 ``.bss`` 段的全局数据清零。

.. code-block:: rust
    :linenos:

    // os/src/main.rs
    fn clear_bss() {
        extern "C" {
            fn sbss();
            fn ebss();
        }
        (sbss as usize..ebss as usize).for_each(|a| {
            unsafe { (a as *mut u8).write_volatile(0) }
        });
    }

在程序内自己进行清零的时候，我们就不用去解析 ELF执行文件格式（此时也没有 ELF 执行文件可供解析）来定位 ``.bss`` 段的位置，而是通过链接脚本 ``linker.ld`` 中给出的全局符号 
``sbss`` 和 ``ebss`` 来确定 ``.bss`` 段的位置。


添加裸机打印相关函数
----------------------------------

与上一节为输出字符实现的代码片段相比，裸机应用的执行环境支持字符输出的代码改动会很小。这里的原因是，二者的区别仅仅是系统调用的参数格式和SBI调用的参数格式的不同。
下面的代码是在基于上节有打印能力的执行环境的基础上做的变动。

.. code-block:: rust

    const SBI_CONSOLE_PUTCHAR: usize = 1;

    pub fn console_putchar(c: usize) {
        syscall(SBI_CONSOLE_PUTCHAR, [c, 0, 0]);
    }

    impl Write for Stdout {
        fn write_str(&mut self, s: &str) -> fmt::Result {
            //sys_write(STDOUT, s.as_bytes());
            for c in s.chars() {
                console_putchar(c as usize);
            }
            Ok(())
        }
    }


可以看到主要就只是把之前的操作系统系统调用改为了SBI调用。然后我们再编译运行试试，

.. code-block:: console

    $ cargo build
    $ rust-objcopy --binary-architecture=riscv64 target/riscv64gc-unknown-none-elf/debug/os --strip-all -O binary target/riscv64gc-unknown-none-elf/debug/os.bin
    $ qemu-system-riscv64 -machine virt -nographic -bios ../bootloader/rustsbi-qemu.bin -device loader,file=target/riscv64gc-unknown-none-elf/debug/os.bin,addr=0x80200000

    [rustsbi] Version 0.1.0
    .______       __    __      _______.___________.  _______..______   __
    |   _  \     |  |  |  |    /       |           | /       ||   _  \ |  |
    |  |_)  |    |  |  |  |   |   (----`---|  |----`|   (----`|  |_)  ||  |
    |      /     |  |  |  |    \   \       |  |      \   \    |   _  < |  |
    |  |\  \----.|  `--'  |.----)   |      |  |  .----)   |   |  |_)  ||  |
    | _| `._____| \______/ |_______/       |__|  |_______/    |______/ |__|

    [rustsbi] Platform: QEMU
    [rustsbi] misa: RV64ACDFIMSU
    [rustsbi] mideleg: 0x222
    [rustsbi] medeleg: 0xb1ab
    [rustsbi] Kernel entry: 0x80200000
    Hello, world!

可以看到，在裸机上输出了 ``Hello, world!`` ，而且qemu正常退出，表示RISC-V计算机也正常关机了。



添加处理异常错误的功能
----------------------------------

接着我们给异常处理函数 ``panic`` 增加显示字符串能力，帮助我们理解出现异常时的代码执行情况。

.. code-block:: rust

    // os/src/main.rs
    #![feature(panic_info_message)]

    #[panic_handler]
    fn panic(info: &PanicInfo) -> ! {
        if let Some(location) = info.location() {
            println!("Panicked at {}:{} {}", location.file(), location.line(), info.message().unwrap());
        } else {
            println!("Panicked: {}", info.message().unwrap());
        }
        shutdown()
    }

我们在 ``main.rs`` 的 ``rust_main`` 函数中调用 ``panic!("It should shutdown!");`` 宏时，整个模拟执行的结果是：

.. code-block:: console

    $ cargo build --release 
    $ rust-objcopy --binary-architecture=riscv64 target/riscv64gc-unknown-none-elf/release/os \
        --strip-all -O binary target/riscv64gc-unknown-none-elf/release/os.bin
    $ qemu-system-riscv64 \
        -machine virt \
        -nographic \
        -bios ../bootloader/rustsbi-qemu.bin \
        -device loader,file=target/riscv64gc-unknown-none-elf/release/os.bin,addr=0x80200000

        [rustsbi] Version 0.1.0
        .______       __    __      _______.___________.  _______..______   __
        |   _  \     |  |  |  |    /       |           | /       ||   _  \ |  |
        |  |_)  |    |  |  |  |   |   (----`---|  |----`|   (----`|  |_)  ||  |
        |      /     |  |  |  |    \   \       |  |      \   \    |   _  < |  |
        |  |\  \----.|  `--'  |.----)   |      |  |  .----)   |   |  |_)  ||  |
        | _| `._____| \______/ |_______/       |__|  |_______/    |______/ |__|

        [rustsbi] Platform: QEMU
        [rustsbi] misa: RV64ACDFIMSU
        [rustsbi] mideleg: 0x222
        [rustsbi] medeleg: 0xb1ab
        [rustsbi] Kernel entry: 0x80200000
        Hello, world!
        Panicked at src/main.rs:95 It should shutdown!

可以看到产生panic的地点在 ``main.rs`` 的第95行，与源码中的实际位置一致！至此，我们完成了第一章的实验内容，

