# 1. 引言
在上一篇介绍CFS调度器设计目标的文章: [CFS 调度器设计](https://mp.weixin.qq.com/s/rfMeN8WvRgq2bKWIVcgpwQ) 中讲到，Linux kernel在6.6版本中引入了EEVDF调度器替换了CFS，本文就讲一下EEVDF调度器。

# 2. CFS设计约束
CFS自从2.6.13版本引入linux内核以来，一直都是使用最广泛的调度器，为什么还要引入EEVDF调度器呢，肯定是有一些地方还不够好。

LWN上有一篇文章[An EEVDF CPU scheduler for Linux](https://lwn.net/Articles/925371/)专门解释了为什么引入EEVDF。Linux 内核的完全公平调度器（CFS）负责为系统内绝大多数进程分配 CPU 运行时间。CFS 在 2007 年随内核 2.6.23 版本合入主线，经过多年持续迭代优化，长期稳定承担调度工作。但 CFS 并非完美，存在不少场景处理效果不佳。内核开发者 Peter Zijlstra 提出的 EEVDF 调度器，有望在改善调度表现的同时，大幅削减 CFS 中大量脆弱、难以维护的启发式判断逻辑。

从名字就能看出，CFS 的核心设计目标是**公平性**：保证系统中每个进程分到应得的 CPU 时间。实现思路是跟踪每个进程已占用的运行时长，优先调度累计 CPU 时间更少的进程；同时每个进程的运行时长会根据自身 nice 优先级做权重换算。简单来说，CFS 本质是加权公平队列调度。

公平性虽然能解决大部分 CPU 调度问题，但调度器还要兼顾大量其他约束：
- **缓存效率**：尽量减少进程跨 CPU 迁移，充分利用 CPU 缓存；
- **CPU 利用率**：有任务待运行时，要保证所有 CPU 核心都处于忙碌状态；
- **功耗管理**：为了延长续航，系统吞吐量有时需要让步于省电策略；
- **异构架构**：大小核架构（性能核 + 能效核）会进一步提升调度复杂度。

其中最亟待优化的痛点，是**延迟需求**处理：
有些进程不需要长时间占用 CPU，但一旦有任务触发，必须立刻获得 CPU；另一些进程需要大量算力，但短暂等待完全可以接受。CFS 缺少专门机制表达进程的延迟诉求：nice 值仅能调整 CPU 时间配额，无法区分延迟敏感程度。
虽然实时调度类可以处理低延迟业务，但实时调度属于特权操作，一旦实时进程长期占用 CPU，会严重拖累整个系统。
业界长期存在 `latency nice` 补丁集试图解决该问题：允许低延迟进程在就绪时插队抢占 CPU。这套方案虽能生效，但 Zijlstra 认为存在更优雅的实现思路，也就是 EEVDF。

# 3. EEVDF调度算法介绍

EEVDF 全称 **Earliest Eligible Virtual Deadline First（最早合格虚拟截止时间优先）**，该算法早在 1995 年就由 Ion Stoica 与 Hussein Abdel-Wahab 在学术论文《[Earliest Eligible Virtual Deadline First : A Flexible
and Accurate Mechanism for Proportional Share
Resource Allocation](https://web.archive.org/web/20260225090858/https://citeseerx.ist.psu.edu/document?doi=805acf7726282721504c8f00575d91ebfd750564&repid=rep1&type=pdf)》中提出。从命名上看它和内核 **截止时间实时调度器（Deadline Scheduler）** 有相似之处，但 EEVDF 不属于实时调度算法，底层实现逻辑完全不同。理解 EEVDF 需要掌握几个基础核心概念。

和 CFS 一致，EEVDF 同样追求 CPU 时间的公平分配。举个例子：单 CPU 上同时运行 5 个同等权重进程，每个进程理应获得 20% 的 CPU 时间；nice 值会调整进程权重，nice 值越低（优先级越高），分到的 CPU 配额越多，其他高 nice 进程相应减少。这部分逻辑和 CFS 没有区别。

假设统计周期为 1 秒，5 个进程理论上各应分到 200ms 运行时间。实际运行很难完全均等：部分进程占用时间超标，部分进程分配不足。EEVDF 为每个进程计算 **滞后值（lag）**：进程理论应得 CPU 时间 减去 实际已运行时间。
- lag > 0：进程少分配 CPU，系统欠该任务运行时长；
- lag < 0：进程占用 CPU 超出自身公平配额。

## 3.1. 合格状态（Eligible）与合格时间
只有 lag ≥ 0 的进程，才被判定为 **合格（eligible）**，具备被调度执行的资格；lag 为负的进程暂时无法参与调度。

不合格的进程会在未来某个时间点，自身理论配额追上实际运行时长，重新恢复合格状态，这个时间点称为 **合格时间（eligible time）**。

lag 的计算是 EEVDF 的核心，整套补丁集很大一部分代码都用于精准维护该数值。即便不完整落地整套 EEVDF，lag 本身也可用于运行队列排序：优先调度 lag 更大的进程，抹平全系统各任务的滞后差值。

## 3.2. 虚拟截止时间（virtual deadline）
另一个核心参数是**虚拟截止时间**，代表进程应当完成本次配额运行的最早时刻。

计算公式：虚拟截止时间 = 合格时间 + 当前分配的时间片

举例：某进程资格时间在未来 20ms，分配时间片 10ms，则它的虚拟截止时间为 30ms。

EEVDF 的调度核心逻辑完全贴合名称：**在所有合格进程中，选择虚拟截止时间最早的任务投入运行**。调度决策同时兼顾两点：公平性（通过 lag 计算资格时间）、任务当前应分配的运行时长。

## 3.3. EEVDF算法
### 3.3.1. lag数值变化
假设有三个 CPU 密集型任务 A、B、C 同时启动，初始所有任务 lag 均为 0：
| 任务 | A | B | C |
| - | - | - | - |
| lag |	0ms | 0ms |	0ms |

全部任务 lag 非负，均为合格状态。调度器先选中 A，分配 30ms 时间片并完整运行完毕。

30ms 真实时间内，三个任务各自应分得 10ms 配额：A 实际占用 30ms，
超额 20ms，lag 变为 - 20ms；B、C 未运行，各亏欠 10ms，lag=10ms。
| 任务 | A | B | C |
| - | - | - | - |
| lag |	-20ms | 10ms |	10ms |

此时 A 不再具备调度资格，调度器从 B、C 二选一。假设 B 拿到 30ms 完整时间片：

过去 30ms 每个任务又新增 10ms 理论配额，B 额外消耗 30ms 运行时长，最终 B 的 lag 变为 - 10ms；C 再新增 10ms 亏欠，lag=20ms。
| 任务 | A | B | C |
| - | - | - | - |
| lag |	-10ms | -10ms |	20ms |

从示例可总结 EEVDF 核心数学特性：**系统内所有任务 lag 总和恒等于 0**。

### 3.3.2. 休眠任务的lag衰减机制

lag 仅对就绪任务有调度意义；休眠一整天的任务不会持续累积巨额 lag（休眠期间本就无 CPU 使用需求）。任务休眠时内核会保留其当前 lag，唤醒后基于该数值继续计算。

如果任务休眠前 CPU 使用超标（lag 为负），唤醒后需要补齐这份 “亏欠”。

但长期休眠任务一直背负过去的负 lag 并不合理：难道休眠一天后，还要为昨天短暂超额占用 CPU 持续受惩罚？业界共识是 lag 最终应当归零，但衰减时机难以界定：
- 若任务一休眠就直接清空 lag：应用可以在时间片末尾短暂休眠，直接抹除负 lag，反复骗取超额 CPU 资源；
- 单纯基于真实 wall-clock 衰减 lag 不可行：lag 与虚拟运行时间强绑定，而虚拟时钟流逝速度动态变化。

**最终方案：基于虚拟运行时间衰减休眠任务 lag**
EEVDF方案采用了**延迟出队（deferred dequeue）**机制：

常规逻辑中任务休眠会直接移出运行队列；新逻辑下，**lag 为负、不具备调度资格的休眠任务会保留在运行队列，仅标记为延迟出队**。

这类任务不会被调度选中，但全局虚拟时间每向前推进，它的 lag 会同步增长；一旦 lag 由负转正变为合格状态，调度器自动将其移出队列。

该机制带来两种效果：
- 短暂休眠的任务无法逃脱负 lag 惩罚，杜绝作弊抢占 CPU；
- 长期休眠任务的负 lag 会随虚拟时间持续衰减，最终债务清零。

补充规则：**正数 lag 会永久保留，直到任务被调度运行消耗掉这份亏欠**。

### 3.3.3. 时间片抢占与自定义时间片接口
前文提到，短时间片任务虚拟截止时间更早、调度优先级更高，但该特性仅在调度器重新选任务时生效。若低延迟短时间片任务中途唤醒，仍需要等待当前长时片任务完整用完时间片才能抢占，延迟无法得到保障。

Zijlstra 的补丁集新增**基于虚拟截止时间的抢占逻辑**：只要新就绪任务的虚拟截止时间早于当前运行任务，即可立即抢占 CPU。

该改动大幅稳定短时片任务的调度时延，代价是长时间批处理任务会被更频繁打断，整体吞吐轻微下降。

#### 3.3.3.1. sched_setattr () 新增自定义时间片能力
遗留内核存在一个短板：非实时进程无法主动向内核声明自身期望的时间片长度。本次补丁补齐该接口：
应用调用 sched_setattr() 系统调用，在 sched_attr 结构体的 sched_runtime 字段传入纳秒级目标时间片。

该字段此前仅用于 SCHED_DEADLINE 实时调度类，现在普通公平调度任务均可使用。
- 设置更短时间片：虚拟截止时间靠前，唤醒后抢占更快、调度频次更高；
- 时间片设置过短：频繁触发上下文切换，反而拖慢任务整体运行速度。

时间片合法取值区间：100 微秒～100 毫秒。

# 4. 总结
EEVDF从6.6开始引入linux内核，到6.12基本全部完成，中间经历了好几轮patch的提交与review，后续可以围绕这些patch提交更好的理解EEVDF的实现细节。

# 5. 引用
[EEVDF Scheduler](https://docs.kernel.org/scheduler/sched-eevdf.html?f_link_type=f_linkinlinenote&flow_extra=eyJpbmxpbmVfZGlzcGxheV9wb3NpdGlvbiI6MCwiZG9jX3Bvc2l0aW9uIjowLCJkb2NfaWQiOiI4MGM4NzkzOGI0NmU1OGMxLTk2ZDYzY2Q3MDExYzU3MjkifQ%3D%3D)

[An EEVDF CPU scheduler for Linux](https://lwn.net/Articles/925371/)

[Earliest Eligible Virtual Deadline First : A Flexible and Accurate Mechanism for Proportional Share Resource Allocation](https://web.archive.org/web/20260225090858/https://citeseerx.ist.psu.edu/document?doi=805acf7726282721504c8f00575d91ebfd750564&repid=rep1&type=pdf)

[Completing the EEVDF scheduler](https://lwn.net/Articles/969062/)