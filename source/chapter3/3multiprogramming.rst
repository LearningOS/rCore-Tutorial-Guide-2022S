管理多道程序
=========================================


而内核为了管理任务，需要维护 **任务** 信息

- 任务运行状态：未初始化、准备执行、正在执行、已退出
- 任务控制块：维护任务状态和任务上下文
- 任务相关系统调用：程序主动暂停 ``sys_yield`` 和主动退出 ``sys_exit``

yield 系统调用
-------------------------------------------------------------------------


.. image:: multiprogramming.png

上图描述了一种多道程序执行的典型情况。其中横轴为时间线，纵轴为正在执行的实体。
开始时，蓝色应用向外设提交了一个请求，外设随即开始工作，
但是它要一段时间后才能返回结果。蓝色应用于是调用 ``sys_yield`` 交出 CPU 使用权，
内核让绿色应用继续执行。一段时间后 CPU 切换回蓝色应用，发现外设仍未返回结果，
于是再次 ``sys_yield`` 。直到再次切换回蓝色应用，外设才处理完请求，于是蓝色应用终于可以向下执行了。

我们还会遇到很多其他需要等待其完成才能继续向下执行的事件，调用 ``sys_yield`` 可以避免等待过程造成的资源浪费。

.. code-block:: rust
    :caption: 第三章新增系统调用（一）

    /// 功能：应用主动交出 CPU 所有权并切换到其他应用。
    /// 返回值：总是返回 0。
    /// syscall ID：124
    fn sys_yield() -> isize;

用户库对应的实现和封装：

.. code-block:: rust
    
    // user/src/syscall.rs

    pub fn sys_yield() -> isize {
        syscall(SYSCALL_YIELD, [0, 0, 0])
    }

    // user/src/lib.rs
    // yield 是 Rust 的关键字
    pub fn yield_() -> isize { sys_yield() }

下文介绍内核应如何实现该系统调用。

任务控制块与任务运行状态
---------------------------------------------------------

任务运行状态暂包括如下几种：

.. code-block:: rust
    :linenos:

    // os/src/task/task.rs

    #[derive(Copy, Clone, PartialEq)]
    pub enum TaskStatus {
        UnInit, // 未初始化
        Ready, // 准备运行
        Running, // 正在运行
        Exited, // 已退出
    }

除了任务状态外，还需要保存任务上下文，一并位于名为 **任务控制块** (Task Control Block) 的数据结构中：

.. code-block:: rust
    :linenos:

    // os/src/task/task.rs

    #[derive(Copy, Clone)]
    pub struct TaskControlBlock {
        pub task_status: TaskStatus,
        pub task_cx: TaskContext,
    }


任务控制块非常重要。在内核中，它就是应用的管理单位。后面的章节我们还会不断向里面添加更多内容。

任务管理器
--------------------------------------

还需要一个全局的任务管理器来管理这些任务控制块：

.. code-block:: rust

    // os/src/task/mod.rs

    pub struct TaskManager {
        num_app: usize,
        inner: UPSafeCell<TaskManagerInner>,
    }

    struct TaskManagerInner {
        tasks: [TaskControlBlock; MAX_APP_NUM],
        current_task: usize,
    }

    unsafe impl Sync for TaskManager {}

这里用到了变量与常量分离的编程风格：字段 ``num_app`` 表示任务管理器管理的应用的数目，它在 ``TaskManager`` 初始化后将保持不变；
而包裹在 ``TaskManagerInner`` 内的任务控制块数组 ``tasks``，以及正在执行的应用编号 ``current_task`` 会在执行过程中变化。

初始化 ``TaskManager`` 的全局实例 ``TASK_MANAGER``：

.. code-block:: rust
    :linenos:

    // os/src/task/mod.rs

    lazy_static! {
        pub static ref TASK_MANAGER: TaskManager = {
            let num_app = get_num_app();
            let mut tasks = [TaskControlBlock {
                task_cx: TaskContext::zero_init(),
                task_status: TaskStatus::UnInit,
            }; MAX_APP_NUM];
            for (i, t) in tasks.iter_mut().enumerate().take(num_app) {
                t.task_cx = TaskContext::goto_restore(init_app_cx(i));
                t.task_status = TaskStatus::Ready;
            }
            TaskManager {
                num_app,
                inner: unsafe {
                    UPSafeCell::new(TaskManagerInner {
                        tasks,
                        current_task: 0,
                    })
                },
            }
        };
    }

- 第 5 行：调用 ``loader`` 子模块提供的 ``get_num_app`` 接口获取链接到内核的应用总数；
- 第 10~12 行：依次对每个任务控制块进行初始化，将其运行状态设置为 ``Ready`` ，并在它的内核栈栈顶压入一些初始化
  上下文，然后更新它的 ``task_cx`` 。一些细节我们会稍后介绍。
- 从第 14 行开始：创建 ``TaskManager`` 实例并返回。

实现 sys_yield 和 sys_exit
----------------------------------------------------------------------------

``sys_yield`` 的实现用到了 ``task`` 子模块提供的 ``suspend_current_and_run_next`` 接口：

.. code-block:: rust

    // os/src/syscall/process.rs

    use crate::task::suspend_current_and_run_next;

    pub fn sys_yield() -> isize {
        suspend_current_and_run_next();
        0
    }

这个接口如字面含义，就是暂停当前的应用并切换到下个应用。

同样， ``sys_exit`` 也改成基于 ``task`` 子模块提供的 ``exit_current_and_run_next`` 接口：

.. code-block:: rust

    // os/src/syscall/process.rs

    use crate::task::exit_current_and_run_next;

    pub fn sys_exit(exit_code: i32) -> ! {
        println!("[kernel] Application exited with code {}", exit_code);
        exit_current_and_run_next();
        panic!("Unreachable in sys_exit!");
    }

它的含义是退出当前的应用并切换到下个应用。在调用它之前我们打印应用的退出信息并输出它的退出码。如果是应用出错也应该调用该接口，不过我们这里并没有实现，有兴趣的读者可以尝试。

那么 ``suspend_current_and_run_next`` 和 ``exit_current_and_run_next`` 各是如何实现的呢？

.. code-block:: rust

    // os/src/task/mod.rs

    pub fn suspend_current_and_run_next() {
        mark_current_suspended();
        run_next_task();
    }

    pub fn exit_current_and_run_next() {
        mark_current_exited();
        run_next_task();
    }

它们都是先修改当前应用的运行状态，然后尝试切换到下一个应用。修改运行状态比较简单，实现如下：

.. code-block:: rust
    :linenos:

    // os/src/task/mod.rs

    fn mark_current_suspended() {
        TASK_MANAGER.mark_current_suspended();
    }

    fn mark_current_exited() {
        TASK_MANAGER.mark_current_exited();
    }

    impl TaskManager {
        fn mark_current_suspended(&self) {
            let mut inner = self.inner.borrow_mut();
            let current = inner.current_task;
            inner.tasks[current].task_status = TaskStatus::Ready;
        }

        fn mark_current_exited(&self) {
            let mut inner = self.inner.borrow_mut();
            let current = inner.current_task;
            inner.tasks[current].task_status = TaskStatus::Exited;
        }
    }

以 ``mark_current_suspended`` 为例。它调用了全局任务管理器 ``TASK_MANAGER`` 的 ``mark_current_suspended`` 方法。其中，首先获得里层 ``TaskManagerInner`` 的可变引用，然后根据其中记录的当前正在执行的应用 ID 对应在任务控制块数组 ``tasks`` 中修改状态。

接下来看看 ``run_next_task`` 的实现：

.. code-block:: rust
    :linenos:

    // os/src/task/mod.rs

    fn run_next_task() {
        TASK_MANAGER.run_next_task();
    }

    impl TaskManager {
        fn run_next_task(&self) {
            if let Some(next) = self.find_next_task() {
                let mut inner = self.inner.borrow_mut();
                let current = inner.current_task;
                inner.tasks[next].task_status = TaskStatus::Running;
                inner.current_task = next;
                let current_task_cx_ptr2 = inner.tasks[current].get_task_cx_ptr2();
                let next_task_cx_ptr2 = inner.tasks[next].get_task_cx_ptr2();
                core::mem::drop(inner);
                unsafe {
                    __switch(
                        current_task_cx_ptr2,
                        next_task_cx_ptr2,
                    );
                }
            } else {
                panic!("All applications completed!");
            }
        }
    }

``run_next_task`` 使用任务管理器的全局实例 ``TASK_MANAGER`` 的 ``run_next_task`` 方法。它会调用 ``find_next_task`` 方法尝试寻找一个运行状态为 ``Ready`` 的应用并返回其 ID 。注意到其返回的类型是 ``Option<usize>`` ，也就是说不一定能够找到，当所有的应用都退出并将自身状态修改为 ``Exited`` 就会出现这种情况，此时 ``find_next_task`` 应该返回 ``None`` 。如果能够找到下一个可运行的应用的话，我们就可以分别拿到当前应用 ``current`` 和即将被切换到的应用 ``next`` 的 ``task_cx_ptr2`` ，然后调用 ``__switch`` 接口进行切换。如果找不到的话，说明所有的应用都运行完毕了，我们可以直接 panic 退出内核。

注意在实际切换之前我们需要手动 drop 掉我们获取到的 ``TaskManagerInner`` 的可变引用。因为一般情况下它是在函数退出之后才会被自动释放，从而 ``TASK_MANAGER`` 的 ``inner`` 字段得以回归到未被借用的状态，之后可以再借用。如果不手动 drop 的话，编译器会在 ``__switch`` 返回，也就是当前应用被切换回来的时候才 drop，这期间我们都不能修改 ``TaskManagerInner`` ，甚至不能读（因为之前是可变借用）。正因如此，我们需要在 ``__switch`` 前提早手动 drop 掉 ``inner`` 。

于是 ``find_next_task`` 又是如何实现的呢？

.. code-block:: rust
    :linenos:

    // os/src/task/mod.rs

    impl TaskManager {
        fn find_next_task(&self) -> Option<usize> {
            let inner = self.inner.borrow();
            let current = inner.current_task;
            (current + 1..current + self.num_app + 1)
                .map(|id| id % self.num_app)
                .find(|id| {
                    inner.tasks[*id].task_status == TaskStatus::Ready
                })
        }
    }

``TaskManagerInner`` 的 ``tasks`` 是一个固定的任务控制块组成的表，长度为 ``num_app`` ，可以用下标 ``0~num_app-1`` 来访问得到每个应用的控制状态。我们的任务就是找到 ``current_task`` 后面第一个状态为 ``Ready`` 的应用。因此从 ``current_task + 1`` 开始循环一圈，需要首先对 ``num_app`` 取模得到实际的下标，然后检查它的运行状态。

.. note:: 

    **Rust 语法卡片：迭代器**

    ``a..b`` 实际上表示左闭右开区间 :math:`[a,b)` ，在 Rust 中，它会被表示为类型 ``core::ops::Range`` ，标准库中为它实现好了 ``Iterator`` trait，因此它也是一个迭代器。

    关于迭代器的使用方法如 ``map/find`` 等，请参考 Rust 官方文档。

我们可以总结一下应用的运行状态变化图：

.. image:: fsm-coop.png

第一次进入用户态
------------------------------------------

在应用真正跑起来之前，需要 CPU 第一次从内核态进入用户态。我们在第二章批处理系统中也介绍过实现方法，只需在内核栈上压入构造好的 Trap 上下文，然后 ``__restore`` 即可。本章的思路大致相同，但是有一些变化。

当一个应用即将被运行的时候，它会被 ``__switch`` 过来。如果它是之前被切换出去的话，那么此时它的内核栈上应该有 Trap 上下文和任务上下文，切换机制可以正常工作。但是如果它是第一次被执行怎么办呢？这就需要它的内核栈上也有类似结构的内容。我们是在创建 ``TaskManager`` 的全局实例 ``TASK_MANAGER`` 的时候来进行这个初始化的。

.. code-block:: rust

    // os/src/task/mod.rs

    for i in 0..num_app {
        tasks[i].task_cx_ptr = init_app_cx(i) as * const _ as usize;
        tasks[i].task_status = TaskStatus::Ready;
    }

当时我们进行了这样的操作。 ``init_app_cx`` 是在 ``loader`` 子模块中定义的：

.. code-block:: rust

    // os/src/loader.rs

    pub fn init_app_cx(app_id: usize) -> &'static TaskContext {
        KERNEL_STACK[app_id].push_context(
            TrapContext::app_init_context(get_base_i(app_id), USER_STACK[app_id].get_sp()),
            TaskContext::goto_restore(),
        )
    }

    impl KernelStack {
        fn get_sp(&self) -> usize {
            self.data.as_ptr() as usize + KERNEL_STACK_SIZE
        }
        pub fn push_context(&self, trap_cx: TrapContext, task_cx: TaskContext) -> &'static mut TaskContext {
            unsafe {
                let trap_cx_ptr = (self.get_sp() - core::mem::size_of::<TrapContext>()) as *mut TrapContext;
                *trap_cx_ptr = trap_cx;
                let task_cx_ptr = (trap_cx_ptr as usize - core::mem::size_of::<TaskContext>()) as *mut TaskContext;
                *task_cx_ptr = task_cx;
                task_cx_ptr.as_mut().unwrap()
            }
        }
    }

这里 ``KernelStack`` 的 ``push_context`` 方法先压入一个和之前相同的 Trap 上下文，再在它上面压入一个任务上下文，然后返回任务上下文的地址。这个任务上下文是我们通过 ``TaskContext::goto_restore`` 构造的：

.. code-block:: rust

    // os/src/task/context.rs

    impl TaskContext {
        pub fn goto_restore() -> Self {
            extern "C" { fn __restore(); }
            Self {
                ra: __restore as usize,
                s: [0; 12],
            }
        }
    }

它只是将任务上下文的 ``ra`` 寄存器设置为 ``__restore`` 的入口地址。这样，在 ``__switch`` 从它上面恢复并返回之后就会直接跳转到 ``__restore`` ，此时栈顶是一个我们构造出来第一次进入用户态执行的 Trap 上下文，就和第二章的情况一样了。

需要注意的是， ``__restore`` 的实现需要做出变化：它 **不再需要** 在开头 ``mv sp, a0`` 了。因为在 ``__switch`` 之后，``sp`` 就已经正确指向了我们需要的 Trap 上下文地址。


在 ``rust_main`` 中我们调用 ``task::run_first_task`` 来开始应用的执行：

.. code-block:: rust
    :linenos:

    // os/src/task/mod.rs

    impl TaskManager {
        fn run_first_task(&self) {
            self.inner.borrow_mut().tasks[0].task_status = TaskStatus::Running;
            let next_task_cx_ptr2 = self.inner.borrow().tasks[0].get_task_cx_ptr2();
            let _unused: usize = 0;
            unsafe {
                __switch(
                    &_unused as *const _,
                    next_task_cx_ptr2,
                );
            }
        }
    }

    pub fn run_first_task() {
        TASK_MANAGER.run_first_task();
    }

这里我们取出即将最先执行的编号为 0 的应用的 ``task_cx_ptr2`` 并希望能够切换过去。注意 ``__switch`` 有两个参数分别表示当前应用和即将切换到的应用的 ``task_cx_ptr2`` ，其第一个参数存在的意义是记录当前应用的任务上下文被保存在哪里，也就是当前应用内核栈的栈顶，这样之后才能继续执行该应用。但在 ``run_first_task`` 的时候，我们并没有执行任何应用， ``__switch`` 前半部分的保存仅仅是在启动栈上保存了一些之后不会用到的数据，自然也无需记录启动栈栈顶的位置。

因此，我们显式声明了一个 ``_unused`` 变量，并将它的地址作为第一个参数传给 ``__switch`` ，这样保存一些寄存器之后的启动栈栈顶的位置将会保存在此变量中。然而无论是此变量还是启动栈我们之后均不会涉及到，一旦应用开始运行，我们就开始在应用的用户栈和内核栈之间开始切换了。这里声明此变量的意义仅仅是为了避免覆盖到其他数据。
