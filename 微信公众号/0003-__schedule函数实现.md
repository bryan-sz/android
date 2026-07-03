# __schedule函数解读
- 本文档以linux内核6.15.0-rc3作为源码基线。

# 函数定义
- __schedule函数定义在kernel/sched/core.c文件第6645行，代码如下：
```
/*
 * __schedule() is the main scheduler function.
 *
 * The main means of driving the scheduler and thus entering this function are:
 *
 *   1. Explicit blocking: mutex, semaphore, waitqueue, etc.
 *
 *   2. TIF_NEED_RESCHED flag is checked on interrupt and userspace return
 *      paths. For example, see arch/x86/entry_64.S.
 *
 *      To drive preemption between tasks, the scheduler sets the flag in timer
 *      interrupt handler sched_tick().
 *
 *   3. Wakeups don't really cause entry into schedule(). They add a
 *      task to the run-queue and that's it.
 *
 *      Now, if the new task added to the run-queue preempts the current
 *      task, then the wakeup sets TIF_NEED_RESCHED and schedule() gets
 *      called on the nearest possible occasion:
 *
 *       - If the kernel is preemptible (CONFIG_PREEMPTION=y):
 *
 *         - in syscall or exception context, at the next outmost
 *           preempt_enable(). (this might be as soon as the wake_up()'s
 *           spin_unlock()!)
 *
 *         - in IRQ context, return from interrupt-handler to
 *           preemptible context
 *
 *       - If the kernel is not preemptible (CONFIG_PREEMPTION is not set)
 *         then at the next:
 *
 *          - cond_resched() call
 *          - explicit schedule() call
 *          - return from syscall or exception to user-space
 *          - return from interrupt-handler to user-space
 *
 * WARNING: must be called with preemption disabled!
 */
static void __sched notrace __schedule(int sched_mode)
{
	struct task_struct *prev, *next;
	/*
	 * On PREEMPT_RT kernel, SM_RTLOCK_WAIT is noted
	 * as a preemption by schedule_debug() and RCU.
	 */
	bool preempt = sched_mode > SM_NONE;
	bool is_switch = false;
	unsigned long *switch_count;
	unsigned long prev_state;
	struct rq_flags rf;
	struct rq *rq;
	int cpu;

	trace_sched_entry_tp(preempt, CALLER_ADDR0);

	cpu = smp_processor_id();
	rq = cpu_rq(cpu);
	prev = rq->curr;

	schedule_debug(prev, preempt);

	if (sched_feat(HRTICK) || sched_feat(HRTICK_DL))
		hrtick_clear(rq);

	local_irq_disable();
	rcu_note_context_switch(preempt);

	/*
	 * Make sure that signal_pending_state()->signal_pending() below
	 * can't be reordered with __set_current_state(TASK_INTERRUPTIBLE)
	 * done by the caller to avoid the race with signal_wake_up():
	 *
	 * __set_current_state(@state)		signal_wake_up()
	 * schedule()				  set_tsk_thread_flag(p, TIF_SIGPENDING)
	 *					  wake_up_state(p, state)
	 *   LOCK rq->lock			    LOCK p->pi_state
	 *   smp_mb__after_spinlock()		    smp_mb__after_spinlock()
	 *     if (signal_pending_state())	    if (p->state & @state)
	 *
	 * Also, the membarrier system call requires a full memory barrier
	 * after coming from user-space, before storing to rq->curr; this
	 * barrier matches a full barrier in the proximity of the membarrier
	 * system call exit.
	 */
	rq_lock(rq, &rf);
	smp_mb__after_spinlock();

	/* Promote REQ to ACT */
	rq->clock_update_flags <<= 1;
	update_rq_clock(rq);
	rq->clock_update_flags = RQCF_UPDATED;

	switch_count = &prev->nivcsw;

	/* Task state changes only considers SM_PREEMPT as preemption */
	preempt = sched_mode == SM_PREEMPT;

	/*
	 * We must load prev->state once (task_struct::state is volatile), such
	 * that we form a control dependency vs deactivate_task() below.
	 */
	prev_state = READ_ONCE(prev->__state);
	if (sched_mode == SM_IDLE) {
		/* SCX must consult the BPF scheduler to tell if rq is empty */
		if (!rq->nr_running && !scx_enabled()) {
			next = prev;
			goto picked;
		}
	} else if (!preempt && prev_state) {
		try_to_block_task(rq, prev, prev_state);
		switch_count = &prev->nvcsw;
	}

	next = pick_next_task(rq, prev, &rf);
	rq_set_donor(rq, next);
picked:
	clear_tsk_need_resched(prev);
	clear_preempt_need_resched();
	rq->last_seen_need_resched_ns = 0;

	is_switch = prev != next;
	if (likely(is_switch)) {
		rq->nr_switches++;
		/*
		 * RCU users of rcu_dereference(rq->curr) may not see
		 * changes to task_struct made by pick_next_task().
		 */
		RCU_INIT_POINTER(rq->curr, next);
		/*
		 * The membarrier system call requires each architecture
		 * to have a full memory barrier after updating
		 * rq->curr, before returning to user-space.
		 *
		 * Here are the schemes providing that barrier on the
		 * various architectures:
		 * - mm ? switch_mm() : mmdrop() for x86, s390, sparc, PowerPC,
		 *   RISC-V.  switch_mm() relies on membarrier_arch_switch_mm()
		 *   on PowerPC and on RISC-V.
		 * - finish_lock_switch() for weakly-ordered
		 *   architectures where spin_unlock is a full barrier,
		 * - switch_to() for arm64 (weakly-ordered, spin_unlock
		 *   is a RELEASE barrier),
		 *
		 * The barrier matches a full barrier in the proximity of
		 * the membarrier system call entry.
		 *
		 * On RISC-V, this barrier pairing is also needed for the
		 * SYNC_CORE command when switching between processes, cf.
		 * the inline comments in membarrier_arch_switch_mm().
		 */
		++*switch_count;

		migrate_disable_switch(rq, prev);
		psi_account_irqtime(rq, prev, next);
		psi_sched_switch(prev, next, !task_on_rq_queued(prev) ||
					     prev->se.sched_delayed);

		trace_sched_switch(preempt, prev, next, prev_state);

		/* Also unlocks the rq: */
		rq = context_switch(rq, prev, next, &rf);
	} else {
		rq_unpin_lock(rq, &rf);
		__balance_callbacks(rq);
		raw_spin_rq_unlock_irq(rq);
	}
	trace_sched_exit_tp(is_switch, CALLER_ADDR0);
}
```
- 前面一大段注释，基本上将__schedule函数的作用和调度器进入方式说清楚了。首先明确，__schedule是主要的调度函数，最终的功能实现在这个函数中。
- 接着说明，驱动调度器并进入该函数的主要方式有：
  - 1. 显式阻塞：mutex、semaphore、waitqueue 等。
  - 2. 在中断路径和用户态返回路径检查 TIF_NEED_RESCHED 标志。例如，参见 arch/x86/entry_64.S。为了在任务之间触发抢占，调度器在定时器中断处理函数sched_tick() 中设置该标志。
  - 3. 唤醒并不会真正导致进入 schedule()。它们只是将任务添加到运行队列而已。现在，如果新添加到运行队列的任务抢占了当前任务，则该唤醒会设置 TIF_NEED_RESCHED，并在最近的可能时机调用 schedule()：
    - 如果内核可抢占 (CONFIG_PREEMPTION=y)：
      - 在系统调用或异常上下文中，在下一个最外层的preempt_enable() 时。（这可能就是 wake_up() 的spin_unlock() 之后！）
      - 在 IRQ 上下文中，返回到可抢占上下文时
    - 如果内核不可抢占（未设置 CONFIG_PREEMPTION）,则在以下情况下：
      - cond_resched() 调用时
      - 显式 schedule() 调用时
      - 从系统调用或异常返回到用户态时
      - 从中断处理返回到用户态时
- 警告：必须在禁止抢占的情况下调用__schedule！

## 关键点说明
1. 调度模式
- 关于__schedule的函数参数，有一个int sched_mode，对应的是如下几种模式：
  - SM_IDLE（-1）：空闲任务调用
  - SM_NONE (0): 普通调度路径
  - SM_PREEMPT(1): 抢占调度
  - SM_RTLOCK_WAIT: PREEMPT_RT下锁等待调度
2. 进入__schedule的准备
- 进入__schedule函数时，需要进行一些准备，保证函数正常运行：
  - 获取本CPU的运行队列：`rq = cpu_rq(cpu)`
  - 记录当前运行任务：`prev = rq->curr`
  - 关中断： `local_irq_disable`
  - 给运行队列加锁： `rq_lock(rq, &rf)`
  - 更新当前CPU运行队列struct rq 的调度时钟 rq->clock：`update_rq_clock(rq)`
  - 记录切换次数：`switch_count = &prev->nivcsw`
3. 当前任务状态处理
- 根据当前进程的状态进行相应处理
  -  读取当前进程状态：`prev_state = READ_ONCE(prev->__state)`
  -  如果是 SM_IDLE 且队列空，则不切换
  -  如果不是抢占且当前任务非运行状态，则调用`try_to_block_task`并更新切换次数
4. 选择下一个任务
- __schedule的核心任务，就是挑选出下一个可运行的任务，调用`next = pick_next_task(rq, prev, &rf);`进行选择
- 寿命的是，pick_next_task是调度器的核心函数，会根据不同的调度类，选择不同的调度类自身实现的pick_next_task函数hook进行下一个可运行任务的选择
- 至此，__schedule函数的任务完成了一半，选出了下一个可运行的任务，剩下的就是切换到选出来的任务上。代码进入到`picked`标记
5. 清除 resched 标志
- 主要是如下几个调用：
```
clear_tsk_need_resched(prev);
clear_preempt_need_resched();
rq->last_seen_need_resched_ns = 0;
``` 
6.  执行切换
- 首先根据选出来的任务判断是否要进行切换 `is_switch = prev != next`,如果prev和next相同，那就省掉切换的步骤，只有在不同的时候，才需要进行切换
- 更新切换计数：
```
rq->nr_switches++
++*switch_count;
```
- 执行切换：`rq = context_switch(rq, prev, next, &rf);`
- 如果不需要切换的话，就说明当前核没有那么繁忙，因此可以考虑进行balance均衡，就会触发一次`__balance_callbacks(rq);`

7. 至此，__schedule函数流程完成，就已经完成了就一个线程切换到另外一个线程的完整过程。

## 完整调用链
根据以上解读，__schedule的核心调用链如下：
- __schedule
  - rq_lock
  - update_rq_clock
  - pick_next_task
  - context_switch
  - __balance_callbacks

# 总结
总体上，__schedule的初步流程就走了一遍，里面最核心的，其实是两个步骤：1、`pick_next_task`选择下一个可运行任务；2、`context_switch`切换到选出来的任务，这就完成了调度器的线程切换的工作。本文只能抛砖引玉，具体的细节和流程，还是需要一行一行代码仔细理解，才能完全清楚。