任务切换
================================


本节我们将见识操作系统的核心机制—— **任务切换** ，
即应用在运行中主动或被动地交出 CPU 的使用权，内核可以选择另一个程序继续执行。
内核需要保证用户程序两次运行期间，任务上下文（如寄存器、栈等）保持一致。

.. note::
    
    函数调用前后，编译器帮我们保证程序上下文一致。

任务切换的设计与实现
---------------------------------

任务切换与上一章提及的 Trap 控制流切换相比，有如下异同：

- 与 Trap 切换不同，它不涉及特权级切换；
- 与 Trap 切换不同，它部分由编译器完成；
- 与 Trap 切换相同，它对应用是透明的。

事实上，任务切换是来自两个不同应用在内核中的 Trap 控制流之间的切换。
当一个应用 Trap 到 S 态 OS 内核中进行进一步处理时，
其 Trap 控制流可以调用一个特殊的 ``__switch`` 函数。
在 ``__switch`` 返回之后，Trap 控制流将继续从调用该函数的位置继续向下执行。
而在调用 ``__switch`` 之后到返回前的这段时间里，
原 Trap 控制流 ``A`` 会先被暂停并被切换出去， CPU 转而运行另一个应用的 Trap 控制流 ``B`` 。
``__switch`` 返回之后，原 Trap 控制流 ``A`` 才会从某一条 Trap 控制流 ``C`` 切换回来继续执行。

从实现的角度讲， ``__switch`` 函数和一个普通的函数之间的核心差别仅仅是它会 **换栈** 。

.. image:: task_context.png

当 Trap 控制流调用 ``__switch`` 函数并进入暂停状态前，让我们考察一下它内核栈上的情况。
如上图所示，准备调用 ``__switch`` 函数之前，
内核栈上从栈底到栈顶分别是保存了应用执行状态的 Trap 上下文、内核在对 Trap 处理过程中留下的调用栈信息。
为了在日后恢复执行原程序，我们需要保存 CPU 的某些寄存器，它们就是 **任务上下文** (Task Context)。

.. code-block:: C

    TaskContext *task_cx_ptr = &task_cx;

``task_cx_ptr`` 保存了任务上下文的地址。

再来看看 ``__switch`` 过程中内核栈的变化：

.. image:: switch-1.png

.. image:: switch-2.png

- 阶段 [1]：在 Trap 控制流 A 调用 ``__switch`` 之前，A 的内核栈上只有 Trap 上下文和 Trap 处理函数的调用栈信息，而 B 是之前被切换出去的，它的栈顶还有额外的一个任务上下文；
- 阶段 [2]：A 在自身的内核栈上分配一块任务上下文的空间在里面保存 CPU 当前的寄存器快照；
- 阶段 [3]：这一步极为关键。将 B 的内核栈栈顶位置赋给 ``sp`` 寄存器，以切换 B 的内核栈。内核栈保存着它迄今为止的执行历史记录，可以说 **换栈也就实现了控制流的切换** 。正是因为这一步， ``__switch`` 才能做到一个函数跨两条控制流执行。
- 阶段 [4]：CPU 从 B 的内核栈栈顶取出任务上下文并恢复寄存器状态，在这之后还要进行函数返回的退栈操作。
- 阶段 [5]：对于 B 而言， ``__switch`` 函数返回，从调用 ``__switch`` 的位置继续向下执行。

从结果来看，我们看到 A 控制流 和 B 控制流的状态发生了互换， A 在保存任务上下文之后进入暂停状态，而 B 则恢复了上下文并在 CPU 上继续执行。

下面我们给出 ``__switch`` 的实现：

.. code-block:: riscv
    :linenos:

    # os/src/task/switch.S

    .altmacro
    .macro SAVE_SN n
        sd s\n, (\n+2)*8(a0)
    .endm
    .macro LOAD_SN n
        ld s\n, (\n+2)*8(a1)
    .endm
        .section .text
        .globl __switch
    __switch:
        # __switch(
        #     current_task_cx_ptr: *mut TaskContext,
        #     next_task_cx_ptr: *const TaskContext
        # )
        # save kernel stack of current task
        sd sp, 8(a0)
        # save ra & s0~s11 of current execution
        sd ra, 0(a0)
        .set n, 0
        .rept 12
            SAVE_SN %n
            .set n, n + 1
        .endr
        # restore ra & s0~s11 of next execution
        ld ra, 0(a1)
        .set n, 0
        .rept 12
            LOAD_SN %n
            .set n, n + 1
        .endr
        # restore kernel stack of next task
        ld sp, 8(a1)
        ret

它的两个参数分别是当前 Trap 控制流和即将被切换到的 Trap 控制流的 ``task_cx_ptr`` ，从 RISC-V 调用规范可知，它们分别通过寄存器 ``a0/a1`` 传入。

阶段 [2] 体现在第 18~26 行，在此期间，内核把寄存器逐个保存在栈上。

``TaskContext`` 里包含的寄存器有：

.. code-block:: rust
    :linenos:

    // os/src/task/context.rs
    #[repr(C)]
    pub struct TaskContext {
        ra: usize,
        sp: usize,
        s: [usize; 12],
    }

``s0~s11`` 是被调用者保存寄存器， ``__switch`` 是用汇编编写的，编译器不会帮我们处理这些寄存器。
保存 ``ra`` 很重要，它记录了 ``__switch`` 函数返回之后应该跳转到哪里继续执行。

我们将这段汇编代码 ``__switch`` 解释为一个 Rust 函数：

.. code-block:: rust
    :linenos:

    // os/src/task/switch.rs

    global_asm!(include_str!("switch.S"));

    extern "C" {
        pub fn __switch(
            current_task_cx_ptr: *mut TaskContext, 
            next_task_cx_ptr: *const TaskContext);
    }

我们会调用该函数来完成切换功能，而不是直接跳转到符号 ``__switch`` 的地址。
因此在调用前后，编译器会帮我们保存和恢复调用者保存寄存器。
  