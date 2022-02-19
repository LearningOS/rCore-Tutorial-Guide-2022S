chapter3练习
=======================================

编程作业
--------------------------------------

任务信息
++++++++++++++++++++++++++

ch3 中，我们的系统已经能够支持多个任务分时轮流运行，我们希望引入一个新的系统调用 ``sys_task_info`` ，定义如下：
.. code-block:: rust

    fn sys_task_info(id: usize, ts: *mut TaskInfo) -> isize

- syscall ID: 410
- 根据任务id查询任务信息，任务信息包括任务id，任务控制块相关信息（任务状态），统计任务系统调用编号到对应次数多映射，统计任务总运行时长。
.. code-block:: rust
    
    struct TaskInfo {
        id: usize,
        status: TaskStatus,
        call: [SyscallInfo; MAX_SYSCALL_NUM],
        time: usize
    }
- 统计系统调用信息采用数组形式对每个系统调用的次数进行统计，相关结构定义如下：
.. code-block:: rust

    struct SyscallInfo {
        id: usize,
        times: usize
    }

- 参数：
    - id: 待查询任务id
    - ts: 待查询任务信息
- 返回值：执行成功返回0，错误返回-1
- 说明：
    - 相关结构已在框架中给出，只需添加逻辑实现功能需求即可
- 提示：
    - 大胆修改已有框架！除了配置文件，你几乎可以随意修改已有框架的内容。
    - 程序运行时间可以通过调用 ``get_time()`` 获取
    - 系统调用次数可以考虑在进入内核态系统调用异常处理函数之后，进入具体系统调用函数之前维护
    - 阅读TaskManager的实现，思考如何维护内核控制块信息（可以在控制块可变部分加入其他需要的信息）
