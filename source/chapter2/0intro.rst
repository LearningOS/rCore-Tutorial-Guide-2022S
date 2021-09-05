引言
================================

本章导读
---------------------------------

..
  chyyuu：有一个ascii图，画出我们做的OS。

**批处理系统** (Batch System) 出现于计算资源匮乏的年代，其核心思想是：
将多个程序打包到一起输入计算机；当一个程序运行结束后，计算机会 *自动* 执行下一个程序。

应用程序难免会出错，如果一个程序的错误导致整个操作系统都无法运行，那就太糟糕了。
*保护* 操作系统不受有意或无意出错的程序破坏的机制被称为 **特权级** (Privilege) 机制，
它实现了用户态和内核态的隔离。

本章在上一章的基础上，让我们的 OS 内核能以批处理的形式一次运行多个应用程序，同时利用特权级机制，
令 OS 不因出错的用户态程序而崩溃。

.. image:: deng-fish.png
   :align: center
   :name: fish-os

实践体验
---------------------------

本章我们的批处理系统将连续运行三个应用程序，放在 ``user/src/bin`` 目录下。

获取本章代码：

.. code-block:: console

   $ git clone https://github.com/zhanghx0905/rCore-Tutorial-2021Autumn.git
   $ cd rCore-Tutorial-2021Autumn
   $ git checkout ch2

在 qemu 模拟器上运行本章代码：

.. code-block:: console

   $ cd os
   $ make run LOG=INFO

批处理系统自动加载并运行了所有的用户程序，尽管某些程序出错了：

.. code-block:: 
   
   [rustsbi] RustSBI version 0.2.0-alpha.4
  .______       __    __      _______.___________.  _______..______   __
  |   _  \     |  |  |  |    /       |           | /       ||   _  \ |  |
  |  |_)  |    |  |  |  |   |   (----`---|  |----`|   (----`|  |_)  ||  |
  |      /     |  |  |  |    \   \       |  |      \   \    |   _  < |  |
  |  |\  \----.|  `--'  |.----)   |      |  |  .----)   |   |  |_)  ||  |
  | _| `._____| \______/ |_______/       |__|  |_______/    |______/ |__|

  [rustsbi] Implementation: RustSBI-QEMU Version 0.0.1
  [rustsbi-dtb] Hart count: cluster0 with 1 cores
  [rustsbi] misa: RV64ACDFIMSU
  [rustsbi] mideleg: ssoft, stimer, sext (0x222)
  [rustsbi] medeleg: ima, ia, bkpt, la, sa, uecall, ipage, lpage, spage (0xb1ab)
  [rustsbi] pmp0: 0x80000000 ..= 0x800fffff (rwx)
  [rustsbi] pmp1: 0x80000000 ..= 0x807fffff (rwx)
  [rustsbi] pmp2: 0x0 ..= 0xffffffffffffff (---)
  [rustsbi] enter supervisor 0x80200000
  [kernel] Hello, world!
  [ INFO] [kernel] num_app = 6
  [ INFO] [kernel] app_0 [0x8020b040, 0x8020f868)
  [ INFO] [kernel] app_1 [0x8020f868, 0x80214090)
  [ INFO] [kernel] app_2 [0x80214090, 0x80218988)
  [ INFO] [kernel] app_3 [0x80218988, 0x8021d160)
  [ INFO] [kernel] app_4 [0x8021d160, 0x80221a68)
  [ INFO] [kernel] app_5 [0x80221a68, 0x80226538)
  [ INFO] [kernel] Loading app_0
  [ERROR] [kernel] PageFault in application, core dumped.
  [ INFO] [kernel] Loading app_1
  [ERROR] [kernel] IllegalInstruction in application, core dumped.
  [ INFO] [kernel] Loading app_2
  [ERROR] [kernel] IllegalInstruction in application, core dumped.
  [ INFO] [kernel] Loading app_3
  [ INFO] [kernel] Application exited with code 1234
  [ INFO] [kernel] Loading app_4
  Hello, world from user mode program!
  [ INFO] [kernel] Application exited with code 0
  [ INFO] [kernel] Loading app_5
  3^10000=5079(MOD 10007)
  3^20000=8202(MOD 10007)
  3^30000=8824(MOD 10007)
  3^40000=5750(MOD 10007)
  3^50000=3824(MOD 10007)
  3^60000=8516(MOD 10007)
  3^70000=2510(MOD 10007)
  3^80000=9379(MOD 10007)
  3^90000=2621(MOD 10007)
  3^100000=2749(MOD 10007)
  Test power OK!
  [ INFO] [kernel] Application exited with code 0
  Panicked at src/batch.rs:68 All applications completed!

本章代码树
-------------------------------------------------

.. code-block::

   ── os
   │   ├── Cargo.toml
   │   ├── Makefile (修改：构建内核之前先构建应用)
   │   ├── build.rs (新增：生成 link_app.S 将应用作为一个数据段链接到内核)
   │   └── src
   │       ├── batch.rss(新增：实现了一个简单的批处理系统)
   │       ├── console.rs
   │       ├── entry.asm
   │       ├── lang_items.rs
   │       ├── link_app.S(构建产物，由 os/build.rs 输出)
   │       ├── linker.ld
   │       ├── logging.rs
   │       ├── main.rs(修改：主函数中需要初始化 Trap 处理并加载和执行应用)
   │       ├── sbi.rs
   │       ├── sync(新增：包装了RefCell，暂时不用关心)
   │       │   ├── mod.rs
   │       │   └── up.rs
   │       ├── syscall(新增：系统调用子模块 syscall)
   │       │   ├── fs.rs(包含文件 I/O 相关的 syscall)
   │       │   ├── mod.rs(提供 syscall 方法根据 syscall ID 进行分发处理)
   │       │   └── process.rs(包含任务处理相关的 syscall)
   │       └── trap(新增：Trap 相关子模块 trap)
   │           ├── context.rs(包含 Trap 上下文 TrapContext)
   │           ├── mod.rs(包含 Trap 处理入口 trap_handler)
   │           └── trap.S(包含 Trap 上下文保存与恢复的汇编代码)
   └── user(新增：应用测例保存在 user 目录下)
      ├── Cargo.toml
      ├── Makefile
      └── src
         ├── bin(基于用户库 user_lib 开发的应用，每个应用放在一个源文件中)
         │   ├── ...
         ├── console.rs
         ├── lang_items.rs
         ├── lib.rs(用户库 user_lib)
         ├── linker.ld(应用的链接脚本)
         └── syscall.rs(包含 syscall 方法生成实际用于系统调用的汇编指令，
                        各个具体的 syscall 都是通过 syscall 来实现的)
   
   cloc os
   -------------------------------------------------------------------------------
   Language                     files          blank        comment           code
   -------------------------------------------------------------------------------
   Rust                            14             62             21            435
   Assembly                         3              9             16            106
   make                             1             12              4             36
   TOML                             1              2              1              9
   -------------------------------------------------------------------------------
   SUM:                            19             85             42            586
   -------------------------------------------------------------------------------


.. 本章代码导读
.. -----------------------------------------------------

.. 相比于上一章的两个简单操作系统，本章的操作系统有两个最大的不同之处，一个是操作系统自身运行在内核态，且支持应用程序在用户态运行，且能完成应用程序发出的系统调用；另一个是能够一个接一个地自动运行不同的应用程序。所以，我们需要对操作系统和应用程序进行修改，也需要对应用程序的编译生成过程进行修改。

.. 首先改进应用程序，让它能够在用户态执行，并能发出系统调用。这其实就是上一章中  :ref:`构建用户态执行环境 <term-print-userminienv>` 小节介绍内容的进一步改进。具体而言，编写多个应用小程序，修改编译应用所需的 ``linker.ld`` 文件来   :ref:`调整程序的内存布局  <term-app-mem-layout>` ，让操作系统能够把应用加载到指定内存地址，然后顺利启动并运行应用程序。

.. 应用程序运行中，操作系统要支持应用程序的输出功能，并还能支持应用程序退出。这需要完成 ``sys_write`` 和 ``sys_exit`` 系统调用访问请求的实现。 具体实现涉及到内联汇编的编写，以及应用与操作系统内核之间系统调用的参数传递的约定。为了让应用在还没实现操作系统之前就能进行运行测试，我们采用了Linux on RISC-V64 的系统调用参数约定。具体实现可参看 :ref:`系统调用 <term-call-syscall>` 小节中的内容。 这样写完应用小例子后，就可以通过  ``qemu-riscv64`` 模拟器进行测试了。  

.. 写完应用程序后，还需实现支持多个应用程序轮流启动运行的操作系统。这里首先能把本来相对松散的应用程序执行代码和操作系统执行代码连接在一起，便于   ``qemu-system-riscv64`` 模拟器一次性地加载二者到内存中，并让操作系统能够找到应用程序的位置。为把二者连在一起，需要对生成的应用程序进行改造，首先是把应用程序执行文件从ELF执行文件格式变成Binary格式（通过 ``rust-objcopy`` 可以轻松完成）；然后这些Binary格式的文件通过编译器辅助脚本 ``os/build.rs`` 转变变成 ``os/src/link_app.S`` 这个汇编文件的一部分，并生成各个Binary应用的辅助信息，便于操作系统能够找到应用的位置。编译器会把把操作系统的源码和 ``os/src/link_app.S`` 合在一起，编译出操作系统+Binary应用的ELF执行文件，并进一步转变成Binary格式。

.. 操作系统本身需要完成对Binary应用的位置查找，找到后（通过 ``os/src/link_app.S`` 中的变量和标号信息完成），会把Binary应用拷贝到 ``user/src/linker.ld`` 指定的物理内存位置（OS的加载应用功能）。在一个应执行完毕后，还能加载另外一个应用，这主要是通过 ``AppManagerInner`` 数据结构和对应的函数 ``load_app`` 和 ``run_next_app`` 等来完成对应用的一系列管理功能。这主要在 :ref:`实现批处理操作系统  <term-batchos>` 小节中讲解。

.. 为了让Binary应用能够启动和运行，操作系统还需给Binary应用分配好执行环境所需一系列的资源。这主要包括设置好用户栈和内核栈（在用户态的应用程序与在内核态的操作系统内核需要有各自的栈，避免应用程序破坏内核的执行），实现Trap 上下文的保存与恢复（让应用能够在发出系统调用到内核态后，还能回到用户态继续执行），完成Trap 分发与处理等工作。由于系统调用和中断处理等内核代码实现涉及用户态与内核态之间的特权级切换细节的汇编代码，与硬件细节联系紧密，所以 :ref:`这部分内容 <term-trap-handle>` 是本章中理解比较困难的地方。如果要了解清楚，需要对涉及到的RISC-V CSR寄存器的功能有清楚的认识。这就需要看看 `RISC-V手册 <http://crva.ict.ac.cn/documents/RISC-V-Reader-Chinese-v2p1.pdf>`_ 的第十章或更加详细的RISC-V的特权级规范文档了。有了上面的实现后，就剩下最后一步，实现 **执行应用程序** 的操作系统功能，其主要实现在 ``run_next_app`` 内核函数中 。完成所有这些功能的实现，“邓式鱼” [#dunk]_ 操作系统就可以正常运行，并能管理多个应用按批处理方式在用户态一个接一个地执行了。

