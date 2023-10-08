# schedule
- linux调度从大的方面来说，分为主动调度和被动调度，而主动调度通常就是进程等待资源或者其他场景下，主动放弃对CPU的占有。这个时候通常就是将进程状态设置为睡眠状态（interruptible或者uninterruptible），然后调用schedule就完成了切换；本文以linux kernel 6.6-rc4为例，解读整个schedule函数；
- 网上关于schedule的函数解读很多了，而且也非常详尽，为什么还要自己再专门写一篇文章记录呢？还是自己专门详细分析和记录，印象才更深刻。
- schedule函数定义在kernel/sched/core.c文件，代码很简短，但是里面知识点不少，就从源码层面一个一个来解读；
```
asmlinkage __visible void __sched schedule(void)
{
	struct task_struct *tsk = current;

	sched_submit_work(tsk);
	do {
		preempt_disable();
		__schedule(SM_NONE);
		sched_preempt_enable_no_resched();
	} while (need_resched());
	sched_update_worker(tsk);
}
EXPORT_SYMBOL(schedule);
```
- __sched这个attribute要说一下，定义在include/linux/sched/debug.h头文件
```
/* Attach to any functions which should be ignored in wchan output. */
#define __sched		__attribute__((__section__(".sched.text")))
```
- 将schedule函数放入到编译后二进制的".sched.text"这个section，作用在注释中说明白了，这个section的函数在wchan输出中都会被忽略；
- schedule函数的返回值是void，传递的参数也是void，这是因为我们调用schedule函数目的就是切换到下一个进程，所以不期望返回值，然后我们也不知道调度器具体会选择哪一个任务作为下一个运行进程，所以自然也不知道传递什么参数；
- schedule首先调用sched_submit_work函数将当前进程的一些work都提交，然后就是关抢占，调用__schedule函数完成真正的进程切换，在__schedule之后的sched_preempt_enable_no_resched必须要等进程下次调度回来才能得到执行了。下面就分各个阶段详细说明。
## sched_submit_work
sched_submit_work函数也定义在kernel/sched/core.c文件，是static修饰的；
```
static inline void sched_submit_work(struct task_struct *tsk)
{
	unsigned int task_flags;

	if (task_is_running(tsk))
		return;

	task_flags = tsk->flags;
	/*
	 * If a worker goes to sleep, notify and ask workqueue whether it
	 * wants to wake up a task to maintain concurrency.
	 */
	if (task_flags & (PF_WQ_WORKER | PF_IO_WORKER)) {
		if (task_flags & PF_WQ_WORKER)
			wq_worker_sleeping(tsk);
		else
			io_wq_worker_sleeping(tsk);
	}

	/*
	 * spinlock and rwlock must not flush block requests.  This will
	 * deadlock if the callback attempts to acquire a lock which is
	 * already acquired.
	 */
	SCHED_WARN_ON(current->__state & TASK_RTLOCK_WAIT);

	/*
	 * If we are going to sleep and we have plugged IO queued,
	 * make sure to submit it to avoid deadlocks.
	 */
	blk_flush_plug(tsk->plug, true);
}
```
- 首先判断task_is_running，其中state成员在task_struct定义中就有如下注释`/* -1 unrunnable, 0 runnable, >0 stopped: */`,所以如果进程本身是runnable，而正常在调用schedule时通常都会将进程状态设置为interruptible或者uninterruptbile，就不用submit_work了（这个地方存疑）；
- 然后根据task的flags确定进程是workqueue中的worker还是

