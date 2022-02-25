chapter5练习
==============================================

编程作业
---------------------------------------------

进程创建
+++++++++++++++++++++++++++++++++++++++++++++

大家一定好奇过为啥进程创建要用 fork + execve 这么一个奇怪的系统调用，就不能直接搞一个新进程吗？思而不学则殆，我们就来试一试！这章的编程练习请大家实现一个完全 DIY 的系统调用 spawn，用以创建一个新进程。

spawn 系统调用定义( `标准spawn看这里 <https://man7.org/linux/man-pages/man3/posix_spawn.3.html>`_ )：

.. code-block:: rust

    fn sys_spawn(path: *const u8) -> isize

- syscall ID: 400
- 功能：新建子进程，使其执行目标程序。
- 说明：成功返回子进程id，否则返回 -1。
- 可能的错误：
    - 无效的文件名。
    - 进程池满/内存不足等资源错误。

TIPS：虽然测例很简单，但提醒读者 spawn **不必** 像 fork 一样复制父进程的地址空间。

stride 调度算法
+++++++++++++++++++++++++++++++++++++++++

ch3 中我们实现的调度算法十分简单。现在我们要为我们的 os 实现一种带优先级的调度算法：stride 调度算法。

算法描述如下:

(1) 为每个进程设置一个当前 stride，表示该进程当前已经运行的“长度”。另外设置其对应的 pass 值（只与进程的优先权有关系），表示对应进程在调度后，stride 需要进行的累加值。

(2) 每次需要调度时，从当前 runnable 态的进程中选择 stride 最小的进程调度。对于获得调度的进程 P，将对应的 stride 加上其对应的步长 pass。

(3) 一个时间片后，回到上一步骤，重新调度当前 stride 最小的进程。

可以证明，如果令 P.pass = BigStride / P.priority 其中 P.priority 表示进程的优先权（大于 1），而 BigStride 表示一个预先定义的大常数，则该调度方案为每个进程分配的时间将与其优先级成正比。证明过程我们在这里略去，有兴趣的同学可以在网上查找相关资料。

其他实验细节：

- stride 调度要求进程优先级 :math:`\geq 2`，所以设定进程优先级 :math:`\leq 1` 会导致错误。
- 进程初始 stride 设置为 0 即可。
- 进程初始优先级设置为 16。

为了实现该调度算法，内核还要增加 set_prio 系统调用

.. code-block:: rust

   // syscall ID：140
   // 设置当前进程优先级为 prio
   // 参数：prio 进程优先级，要求 prio >= 2
   // 返回值：如果输入合法则返回 prio，否则返回 -1
   fn sys_set_priority(prio: isize) -> isize;


.. attention::

    为了让大家能在本编程作业中使用 ``Vec`` 等数据结构，我们利用第三方库 ``buddy_system_allocator`` 为大家实现了堆内存分配器，相关代码位于 ``heap_alloc`` 模块。

    背景知识： `Rust 中的动态内存分配 <https://rcore-os.github.io/rCore-Tutorial-Book-v3/chapter4/1rust-dynamic-allocation.html>`_

实验要求
+++++++++++++++++++++++++++++++++++++++++++++
- 实现分支：ch5。
- 实验目录请参考 ch3。
- 通过所有测例。

  在 os 目录下 ``make run TEST=1`` 加载所有测例， ``test_usertest`` 打包了所有你需要通过的测例，你也可以通过修改这个文件调整本地测试的内容。 ``make run TEST=2`` 检查 ``set_priority`` 语义的正确性， ``make run TEST=3`` 检查 stride 调度算法是否满足公平性要求，
  六个子程序运行的次数应该大致与其优先级呈正比，测试通过标准是 :math:`\max{\frac{runtimes}{prio}}/ \min{\frac{runtimes}{prio}} < 1.5`.

  CI 的原理是用 ``test_usertest`` 替代 ``initproc`` ，使内核在所有测例执行完后直接退出。

  从本章开始，你的内核必须前向兼容，能通过前一章的所有测例。

.. note::

    利用 ``git cherry-pick`` 系列指令，能方便地将前一章分支 commit 移植到本章分支。

问答作业
--------------------------------------------

(1) fork + exec 的一个比较大的问题是 fork 之后的内存页/文件等资源完全没有使用就废弃了，针对这一点，有什么改进策略？

(2) [选做，不占分]其实使用了题(1)的策略之后，fork + exec 所带来的无效资源的问题已经基本被解决了，但是今年来 fork 还是在被不断的批判，那么到底是什么正在"杀死"fork？可以参考 `论文 <https://www.microsoft.com/en-us/research/uploads/prod/2019/04/fork-hotos19.pdf>`_ 。

(3) 请阅读下列代码，并分析程序的输出，假定不发生运行错误，不考虑行缓冲：

    .. code-block:: c

      int main(){
          int val = 2;

          printf("%d", 0);
          int pid = fork();
          if (pid == 0) {
              val++;
              printf("%d", val);
          } else {
              val--;
              printf("%d", val);
              wait(NULL);
          }
          val++;
          printf("%d", val);
          return 0;
      }


    如果 fork() 之后主程序先运行，则结果如何？如果 fork() 之后 child 先运行，则结果如何？


(4) 请阅读下列代码，分析程序的输出 ``A`` 的数量：( 已知 ``&&`` 的优先级比 ``||`` 高)

    .. code-block:: c

      int main() {
          fork() && fork() && fork() || fork() && fork() || fork() && fork();
          printf("A");
          return 0;
      }

    [选做，不占分] 更进一步，如果给出一个 ``&&`` ``||`` 的序列，如何设计一个程序来得到答案？

(5) stride 算法深入

   stride 算法原理非常简单，但是有一个比较大的问题。例如两个 pass = 10 的进程，使用 8bit 无符号整形储存 stride， p1.stride = 255, p2.stride = 250，在 p2 执行一个时间片后，理论上下一次应该 p1 执行。

   - 实际情况是轮到 p1 执行吗？为什么？

   我们之前要求进程优先级 >= 2 其实就是为了解决这个问题。可以证明，**在不考虑溢出的情况下**, 在进程优先级全部 >= 2 的情况下，如果严格按照算法执行，那么 STRIDE_MAX – STRIDE_MIN <= BigStride / 2。

   - 为什么？尝试简单说明（不要求严格证明）。

   - 已知以上结论，**考虑溢出的情况下**，可以为 Stride 设计特别的比较器，让 BinaryHeap<Stride> 的 pop 方法能返回真正最小的 Stride。补全下列代码中的 ``partial_cmp`` 函数，假设两个 Stride 永远不会相等。

   .. code-block:: rust

     use core::cmp::Ordering;

     struct Stride(u64);

     impl PartialOrd for Stride {
         fn partial_cmp(&self, other: &Self) -> Option<Ordering> {
             // ...
         }
     }

     impl PartialEq for Stride {
         fn eq(&self, other: &Self) -> bool {
             false
         }
     }

   TIPS: 使用 8 bits 存储 stride, BigStride = 255, 则: ``(125 < 255) == false``, ``(129 < 255) == true``.

报告要求
------------------------------------------------------------

- 简单总结你实现的功能（200字以内，不要贴代码）。
- 完成问答题，看清楚选做。
- (optional) 你对本次实验设计及难度/工作量的看法，以及有哪些需要改进的地方，欢迎畅所欲言。
