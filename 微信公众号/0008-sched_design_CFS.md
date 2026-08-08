# 引言
- 本文档主要是讲清楚CFS调度类的设计目的和方法，因此本次不涉及到代码解读。
- 在[pick_next_entity函数解读](https://mp.weixin.qq.com/s/QyEOGLp3uRswNnls60rOrA)中讲到`pick_eevdf`，就涉及到了EEVDF调度器，EEVDF在linux-6.6开始引入内核与CFS并列，到6.10完成完全体并替换掉了CFS。而CFS从2007年linux-2.6.23开始引入内核，到最终2024年linux-6.10被EEVDF替换，中间长达17年，并且是国内甚至全世界Linux大爆发甚至IT行业大爆发的年代，因此CFS的名声无人不知，也是很多linux内核开发人员绕不过去的门槛，因此，即便现在已经进入了EEVDF的年代，对于CFS还是有了解的必要，只有理解了CFS，才能更好的理解EEVDF。
- 最好的CFS文档，就是Linux官方的文档：[CFS Scheduler](https://docs.kernel.org/scheduler/sched-design-CFS.html)，将这篇文章翻译成中文，更好更方便理解。
- Linux内核也有专门的translation工作，因此，也可以到[完全公平调度器](https://docs.kernel.org/translations/zh_CN/scheduler/sched-design-CFS.html)这里查看官方的中文翻译。不过，据我对比，这个还是比英文版本更新要晚一些，信息也没有完全同步，所以还是建议英文原版。


# CFS 调度器

> 原文：[CFS Scheduler](https://docs.kernel.org/scheduler/sched-design-CFS.html)

## 1. 概述

CFS 是“完全公平调度器”（Completely Fair Scheduler）的缩写，是由 Ingo Molnar 实现并合入 Linux 2.6.23 的“桌面”进程调度器。它最初合入内核时，取代了之前 vanilla 调度器中用于处理 `SCHED_OTHER` 交互性的代码。如今，CFS 正在逐步让位于 EEVDF，相关文档参见 [EEVDF Scheduler](https://docs.kernel.org/scheduler/sched-eevdf.html)。

CFS 设计的 80% 可以用一句话概括：CFS 在真实硬件上模拟一个“理想的、精确的多任务 CPU”。

“理想的多任务 CPU”是一种不存在的 CPU，它拥有 100% 的物理处理能力，可以让每个任务以完全相同的速度并行运行，每个任务获得 `1/nr_running` 的处理能力。例如，有两个任务运行时，它们各自获得 50% 的物理处理能力，也就是实际上并行运行。

在真实硬件上，我们一次只能运行一个任务，因此需要引入“虚拟运行时间”（virtual runtime）的概念。一个任务的虚拟运行时间表示：按照上面所说的理想多任务 CPU，该任务的下一个时间片应该何时开始执行。实际上，任务的虚拟运行时间就是它的实际运行时间按照当前运行任务总数进行归一化后的结果。

## 2. 少量实现细节

在 CFS 中，虚拟运行时间通过每个任务的 `p->se.vruntime` 值来表示和跟踪，单位是纳秒。这样就可以精确地为任务标记时间，并衡量任务“应该获得的 CPU 时间”。

一个小细节是：在“理想”硬件上，任意时刻所有任务的 `p->se.vruntime` 值都应该相同。也就是说，所有任务会同时执行，并且没有任务会偏离它在“理想”CPU 时间中的应得份额。

CFS 基于 `p->se.vruntime` 进行任务选择，因此逻辑非常简单：它总是尝试运行 `p->se.vruntime` 最小的任务，也就是目前运行时间最少的任务。CFS 始终尽可能接近“理想的多任务硬件”，在所有可运行任务之间分配 CPU 时间。

CFS 其余的大部分设计都直接源于这个非常简单的概念，只是在此基础上增加了 nice 级别、多处理器支持，以及用于识别睡眠任务的各种算法变体。

## 3. 红黑树

CFS 的设计相当激进：它不再使用旧的运行队列数据结构，而是使用一棵按时间排序的红黑树，构建任务未来执行的“时间线”。因此，它没有前一个 vanilla 调度器以及 RSDL/SD 调度器所受影响的“数组切换”痕迹。

CFS 还维护 `rq->cfs.min_vruntime`，这是一个单调递增的值，用于跟踪运行队列中所有任务的最小虚拟运行时间。系统完成的工作总量通过 `min_vruntime` 进行跟踪；这个值还用于尽可能把新激活的调度实体放到树的左侧。

就绪队列总负载由 rq->cfs.load 统计，数值等于队列内全部任务权重之和。

CFS 维护一棵按时间排序的红黑树，所有可运行任务都按照 `p->se.vruntime` 这一键值排序。CFS 选择树中最“左侧”的任务并运行它。随着系统不断向前运行，已经执行过的任务会逐渐被放到树的右侧，使每个任务都有机会成为“最左侧任务”，并在一个确定的时间范围内获得 CPU。

总结来说，CFS 的工作方式是：先让一个任务运行一小段时间；当任务主动调度，或者调度器时钟Tick到来时，就对该任务的 CPU 使用情况进行“记账”，把它刚刚使用物理 CPU 的那段时间加到 `p->se.vruntime` 上。当 `p->se.vruntime` 增长到足够大，使另一个任务成为它维护的按时间排序红黑树中的“最左侧任务”时，调度器就选择这个新的最左侧任务，并抢占当前任务。

这里还会保留一小段相对于最左侧任务的“粒度”（granularity）距离，以避免任务切换过于频繁、破坏缓存。

## 4. CFS 的一些特性

CFS 使用纳秒级粒度进行记账，不依赖任何 jiffies 或其他 HZ 细节。因此，CFS 调度器不像之前的调度器那样具有“时间片”的概念，也完全没有启发式算法。它只有一个核心可调参数：

```text
/sys/kernel/debug/sched/base_slice_ns
```

该参数可以把调度器从适合“桌面”工作负载的模式，也就是低延迟模式，调节到适合“服务器”工作负载的模式，也就是良好的批处理模式。默认值适合桌面工作负载。`SCHED_BATCH` 也由 CFS 调度器模块处理。

如果 `CONFIG_HZ` 导致 `base_slice_ns < TICK_NSEC`，那么 `base_slice_ns` 对工作负载几乎不会产生影响。

由于自身的设计，CFS 调度器不容易受到当前针对传统调度器启发式算法的各种“攻击”影响：`fiftyp.c`、`thud.c`、`chew.c`、`ring-test.c` 和 `massive_intr.c` 都可以正常工作，不会影响交互性，并能产生预期行为。

与之前的 vanilla 调度器相比，CFS 对 nice 级别和 `SCHED_BATCH` 的处理更强：这两类工作负载会被更加彻底地隔离。

SMP 负载均衡也经过了重新设计和清理：负载均衡代码不再假定需要遍历运行队列，而是使用调度模块提供的迭代器。因此，均衡代码也简化了不少。

## 5. 调度策略

CFS 实现了三种调度策略：

- `SCHED_NORMAL`（传统上称为 `SCHED_OTHER`）：普通任务使用的调度策略。
- `SCHED_BATCH`：相比普通任务，它几乎不会那么频繁地被抢占，因此任务可以运行更长时间，更好地利用缓存，但代价是交互性下降。这种策略适合批处理任务。
- `SCHED_IDLE`：它的优先级甚至弱于 nice 19，但并不是真正的 idle timer 调度器，以避免出现会导致机器死锁的优先级反转问题。

`SCHED_FIFO` 和 `SCHED_RR` 在 `sched/rt.c` 中实现，并遵循 POSIX 的规定。

来自 util-linux-ng 2.13.1.1 的 `chrt` 命令可以设置上述所有策略，但 `SCHED_IDLE` 除外。

## 6. 调度类

新的 CFS 调度器采用“调度类”（Scheduling Classes）设计，引入了一个可扩展的调度器模块层次结构。这些模块封装了调度策略的细节，由调度器核心负责处理，而核心代码不需要对各个模块作过多假设。

`sched/fair.c` 实现了上文介绍的 CFS 调度器。

`sched/rt.c` 实现了 `SCHED_FIFO` 和 `SCHED_RR` 语义，并且比之前的 vanilla 调度器更简单。它使用 100 个运行队列，对应 100 个实时优先级，而不是之前调度器使用的 140 个；同时它不需要 expired 数组。

调度类通过 `sched_class` 结构体实现。该结构体包含一组钩子函数，每当发生相关事件时，调度器就会调用这些函数。

下面是部分钩子函数：

- `enqueue_task(...)`

	当任务进入可运行状态时调用。它会把调度实体（任务）放入红黑树，并增加 `nr_running` 变量。

- `dequeue_task(...)`

	当任务不再可运行时调用，用于把对应的调度实体从红黑树中移除，并减少 `nr_running` 变量。

- `yield_task(...)`

	当前运行任务通过该函数主动让出 CPU。该函数会把当前任务在运行队列中的位置向后移动，使其他可运行任务优先得到调度。

- `wakeup_preempt(...)`

	检查刚刚进入可运行状态的任务是否应该抢占当前运行任务。

- `pick_next_task(...)`

	选择下一个有资格运行且最合适的任务。

- `set_next_task(...)`

	当任务改变调度类、改变任务组或被调度时调用。

- `task_tick(...)`

	该函数主要由时钟Tick函数调用，并可能导致进程切换。它驱动运行中的抢占机制。

## 7. CFS 的组调度扩展

通常，调度器针对单个任务工作，并努力为每个任务提供公平的 CPU 时间。但有时我们希望将任务分组，并为每个任务组提供公平的 CPU 时间。例如，可以先为系统中的每个用户公平地分配 CPU 时间，再为属于该用户的每个任务分配时间。

`CONFIG_CGROUP_SCHED` 正是为了实现这一目标。它允许将任务分组，并在这些任务组之间公平地分配 CPU 时间。

`CONFIG_RT_GROUP_SCHED` 允许对实时任务，也就是 `SCHED_FIFO` 和 `SCHED_RR` 任务进行分组。

`CONFIG_FAIR_GROUP_SCHED` 允许对 CFS 任务，也就是 `SCHED_NORMAL` 和 `SCHED_BATCH` 任务进行分组。

这些选项要求定义 `CONFIG_CGROUPS`，并允许管理员通过 `cgroup` 伪文件系统创建任意任务组。关于该文件系统的更多信息，请参见 [Control Groups](https://docs.kernel.org/admin-guide/cgroup-v1/cgroups.html)。

定义 `CONFIG_FAIR_GROUP_SCHED` 后，通过伪文件系统创建的每个任务组都会生成一个 `cpu.shares` 文件。下面的示例展示了如何创建任务组，以及如何使用 `cgroups` 伪文件系统修改它们的 CPU 份额：

```sh
# mount -t tmpfs cgroup_root /sys/fs/cgroup
# mkdir /sys/fs/cgroup/cpu
# mount -t cgroup -ocpu none /sys/fs/cgroup/cpu
# cd /sys/fs/cgroup/cpu

# mkdir multimedia       # 创建 multimedia 任务组
# mkdir browser          # 创建 browser 任务组

# 将 multimedia 组配置为获得 browser 组两倍的 CPU 带宽
# echo 2048 > multimedia/cpu.shares
# echo 1024 > browser/cpu.shares

# 启动 firefox，并将其移动到 browser 组
# firefox &
# echo <firefox_pid> > browser/tasks

# 启动 gmplayer（或你喜欢的其他播放器）
# echo <movie_player_pid> > multimedia/tasks
```

## 其他链接

- [Linux 内核文档](https://docs.kernel.org/index.html)
- [CFS调度器文档](https://docs.kernel.org/_sources/scheduler/sched-design-CFS.rst.txt)
- [官方中文翻译：完全公平调度器](https://docs.kernel.org/translations/zh_CN/scheduler/sched-design-CFS.html)