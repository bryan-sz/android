# schedule函数解读
- 本文档以linux内核6.15.0-rc3作为源码基线。

# 函数定义
- schedule函数位于kernel/sched/core.c文件第6850行，函数定义如下：
```
asmlinkage __visible void __sched schedule(void)
{
	struct task_struct *tsk = current;

#ifdef CONFIG_RT_MUTEXES
	lockdep_assert(!tsk->sched_rt_mutex);
#endif

	if (!task_is_running(tsk))
		sched_submit_work(tsk);
	__schedule_loop(SM_NONE);
	sched_update_worker(tsk);
}
EXPORT_SYMBOL(schedule);
```
- 这个函数很短，下面将分别从几方面进行解读：

## asmlinkage
- 这个关键字定义在include\linux\linkage.h文件中第15行，在arm架构和内核c语言环境下，就是一个空定义，可以忽略。
```
#ifdef __cplusplus
#define CPP_ASMLINKAGE extern "C"
#else
#define CPP_ASMLINKAGE
#endif

#ifndef asmlinkage
#define asmlinkage CPP_ASMLINKAGE
#endif
```

## __visible
- 这个关键字定义在include\linux\compiler_attributes.h文件第143行，可以看到
```
/*
 * Optional: not supported by clang
 *
 *   gcc: https://gcc.gnu.org/onlinedocs/gcc/Common-Function-Attributes.html#index-externally_005fvisible-function-attribute
 */
#if __has_attribute(__externally_visible__)
# define __visible                      __attribute__((__externally_visible__))
#else
# define __visible
#endif
```
- 可以看到，使用的是gcc的__externally_visible__这个attribute；
- __visible 是 Linux 内核里用来强制符号对外可见、防止被编译器 / 链接器优化删掉的 GCC 属性宏，告诉编译器 / 链接器：这个函数 / 变量会被外部（尤其汇编或其他编译单元）引用，不要把它当成 “未使用” 而优化掉、也不要把它变成 static。核心作用就是强制保留符号、保持全局可见性，可被汇编 / 其他模块正常链接。

## __sched
- 这个关键字定义在include\linux\sched\debug.h文件第44行，可以看到：
```
/* Attach to any functions which should be ignored in wchan output. */
#define __sched		__section(".sched.text")
```
- 意思就是编译时，会将这个函数放入到vmlinux二进制的.sched.text这个section，而不是默认的section。
- 为什么要放到.sched.text这个section呢，注释写的很清楚了，就是在wchan的输出中将这个函数ignore。关于wchan，后续其他章节再展开（埋个雷）。

# 完整调用链
- schedule
  - task_is_running
    - sched_submit_work
  - __schedule_loop
    - preempt_disable
    - __schedule
      - update_rq_clock
      - pick_next_task
      - clear_tsk_need_resched
      - clear_preempt_need_resched
      - context_switch
    - sched_preempt_enable_no_resched
  - sched_update_worker

# 函数实现
## current
- 首先获取当前current线程；`struct task_struct *tsk = current;`

## lockdep死锁检查
- 具体代码实现：
```
#ifdef CONFIG_RT_MUTEXES
	lockdep_assert(!tsk->sched_rt_mutex);
#endif
```
- 这一段代码是针对RT_MUTEX的死锁检查，与调度逻辑本身关系不大，可以暂时先不展开；

## 提交work
- 对应代码是如下两行：
```
	if (!task_is_running(tsk))
		sched_submit_work(tsk);
```
- 首先判断当前任务状态，如果是TASK_RUNNING状态，就不需要提交对应的任务，只有在非TASK_RUNNING状态时，才需要调用sched_submit_work；
- sched_submit_work是针对如果current是worker或者io worker的情形，需要进入sleep状态时对应的workqueue需要进行的状态维护；

## __schedule_loop
- 这个是核心函数__schedule的包装函数，具体定义如下：
```
static __always_inline void __schedule_loop(int sched_mode)
{
	do {
		preempt_disable();
		__schedule(sched_mode);
		sched_preempt_enable_no_resched();
	} while (need_resched());
}
```
- 首先关抢占，调用__schedule进行调度，然后恢复抢占；
- do while的模式保证至少执行一次__schedule进行调度；
- 也存在多次调度的可能，所以需要使用循环来进行多次调度，多次调度可能的典型场景如下：
  - 高优先级 RT 任务持续就绪；
  - 调度过程中中断 / 软中断唤醒了新任务，置位 TIF_NEED_RESCHED；
  - 分时调度中时间片耗尽连锁触发重调度。
- 总之，只要TIF_NEED_RESCHED被置位，就说明有调度请求，需要继续进行调度；

### 调度模式
- schedule调用__schedule_loop时，传入的是SM_NONE这个调度模式，表示默认，其余几个调度模式定义如下：
```
/*
 * Constants for the sched_mode argument of __schedule().
 *
 * The mode argument allows RT enabled kernels to differentiate a
 * preemption from blocking on an 'sleeping' spin/rwlock.
 */
#define SM_IDLE			(-1)
#define SM_NONE			0
#define SM_PREEMPT		1
#define SM_RTLOCK_WAIT		2
```
- 从注释就能看出来，这和实时RT相关的特性引入，具体是https://www.spinics.net/lists/linux-tip-commits/msg58350.html 这次提交，主要视为PREEMPT-RT做的，具体每个调度模式如下：

| 宏名	| 值	| 调度类型 |	典型场景 |	主要适用内核 |
| - | - | - | - | - | 
| SM_IDLE |	-1 | 空闲调度 |	无就绪任务，切换到 idle 线程 |	所有内核 |
| SM_NONE |	0 |	普通主动调度 |	任务主动睡眠、等待队列、IO 阻塞	| 标准内核 + RT 内核通用 |
| SM_PREEMPT |	1 |	被动抢占调度 | 被高优先级任务打断、时间片到期 | 抢占内核 / RT 内核 |
| SM_RTLOCK_WAIT |	2 |	RT 睡眠锁等待调度 |	拿不到 RT 睡眠自旋锁 /rwlock，主动让出 CPU | 	仅 PREEMPT-RT 实时内核 |

- SM_NONE用的最多，总结如下：
    - 含义：无特殊标记，常规主动调度
    - 场景：任务主动放弃 CPU，而非被抢占。
    - 常见路径：
      - 任务调用 schedule() / schedule_timeout() 主动睡眠、等待队列阻塞；
      - IO 阻塞、信号等待、普通互斥锁等待；
    - 内核行为：
      - 按标准进程阻塞逻辑处理，不标记为抢占、不标记为 RT 锁等待，走通用调度分支。

## sched_update_worker
- 这是schedule函数的调用的最后一个函数，这个也是在下一次调度切换回来后才会执行的函数，用于在调度返回时更新 I/O 工作线程状态，避免死锁和饥饿。

# 总结
&emsp;&emsp;总体上，schedule的初步流程就走了一遍，核心的__schedule留到下一篇再讲。作为内核调度的入口函数，本文只能抛砖引玉，具体的细节和流程，还是需要一行一行代码仔细理解，才能完全清楚。