# 作用
- 多核系统上，我们经常需要给其他核发送通知，通常叫做IPI，通知其他核做一些事情，这个时候就可以用上smp_call_function这个函数了，这个功能从这个函数名字也能看出来。
# 原型
```
/**
 * smp_call_function(): Run a function on all other CPUs.
 * @func: The function to run. This must be fast and non-blocking.
 * @info: An arbitrary pointer to pass to the function.
 * @wait: If true, wait (atomically) until function has completed
 *        on other CPUs.
 *
 * Returns 0.
 *
 * If @wait is true, then returns once @func has returned; otherwise
 * it returns just before the target cpu calls @func.
 *
 * You must not call this function with disabled interrupts or from a
 * hardware interrupt handler or from a bottom half handler.
 */
void smp_call_function(smp_call_func_t func, void *info, int wait)
{
	preempt_disable();
	smp_call_function_many(cpu_online_mask, func, info, wait);
	preempt_enable();
}
EXPORT_SYMBOL(smp_call_function);
```
- 函数定义在kernel/smp.c文件中
- 从函数的注释可以看出来，传递的三个参数，func在其他核执行的函数，info是传递给func的参数，而wait就是是否等待func在其他核执行完成；
- 注意，对func函数是有要求的，第一是要简短，能够很快的执行完成，第二是func函数不能阻塞，要是在执行过程中阻塞了被调度出去，然后wait参数还是true的话，这个核就等死了；
- 进来首先是调用preempt_disable关抢占，最后在执行完smp_call_function_many后preempt_enable再打开抢占，保护smp_call_function_many执行的临界区；
- cpu_online_mask是__cpu_online_mask的包装，而__cpu_online_mask是一个全局变量位图，由__cpu_online_mask在cpu上线时设置对应的bit位；
- 由此可见，smp_call_function是会给全部online核发送IPI，让对方执行func函数；
# smp_call_function_many
```
/**
 * smp_call_function_many(): Run a function on a set of CPUs.
 * @mask: The set of cpus to run on (only runs on online subset).
 * @func: The function to run. This must be fast and non-blocking.
 * @info: An arbitrary pointer to pass to the function.
 * @wait: Bitmask that controls the operation. If %SCF_WAIT is set, wait
 *        (atomically) until function has completed on other CPUs. If
 *        %SCF_RUN_LOCAL is set, the function will also be run locally
 *        if the local CPU is set in the @cpumask.
 *
 * If @wait is true, then returns once @func has returned.
 *
 * You must not call this function with disabled interrupts or from a
 * hardware interrupt handler or from a bottom half handler. Preemption
 * must be disabled when calling this function.
 */
void smp_call_function_many(const struct cpumask *mask,
			    smp_call_func_t func, void *info, bool wait)
{
	smp_call_function_many_cond(mask, func, info, wait * SCF_WAIT, NULL);
}
EXPORT_SYMBOL(smp_call_function_many);
```
- smp_call_function_many比smp_call_function多了一个cpumask的参数，比smp_call_function的功能更通用一点，相比smp_call_function一股脑让所有上线的核执行func函数，smp_call_function_many可以指定需要哪些核执行func函数；
- 真正的功能实现，还是调用smp_call_function_many_cond实现的；
# smp_call_function_many_cond
```
static void smp_call_function_many_cond(const struct cpumask *mask,
					smp_call_func_t func, void *info,
					unsigned int scf_flags,
					smp_cond_func_t cond_func)
{
	int cpu, last_cpu, this_cpu = smp_processor_id();
	struct call_function_data *cfd;
	bool wait = scf_flags & SCF_WAIT;
	int nr_cpus = 0;
	bool run_remote = false;
	bool run_local = false;

	lockdep_assert_preemption_disabled();

	/*
	 * Can deadlock when called with interrupts disabled.
	 * We allow cpu's that are not yet online though, as no one else can
	 * send smp call function interrupt to this cpu and as such deadlocks
	 * can't happen.
	 */
	if (cpu_online(this_cpu) && !oops_in_progress &&
	    !early_boot_irqs_disabled)
		lockdep_assert_irqs_enabled();

	/*
	 * When @wait we can deadlock when we interrupt between llist_add() and
	 * arch_send_call_function_ipi*(); when !@wait we can deadlock due to
	 * csd_lock() on because the interrupt context uses the same csd
	 * storage.
	 */
	WARN_ON_ONCE(!in_task());

	/* Check if we need local execution. */
	if ((scf_flags & SCF_RUN_LOCAL) && cpumask_test_cpu(this_cpu, mask))
		run_local = true;

	/* Check if we need remote execution, i.e., any CPU excluding this one. */
	cpu = cpumask_first_and(mask, cpu_online_mask);        
	if (cpu == this_cpu)
		cpu = cpumask_next_and(cpu, mask, cpu_online_mask);
	if (cpu < nr_cpu_ids)
		run_remote = true;

	if (run_remote) {
		cfd = this_cpu_ptr(&cfd_data);
		cpumask_and(cfd->cpumask, mask, cpu_online_mask);
		__cpumask_clear_cpu(this_cpu, cfd->cpumask);

		cpumask_clear(cfd->cpumask_ipi);
		for_each_cpu(cpu, cfd->cpumask) {
			call_single_data_t *csd = per_cpu_ptr(cfd->csd, cpu);

			if (cond_func && !cond_func(cpu, info)) {
				__cpumask_clear_cpu(cpu, cfd->cpumask);
				continue;
			}

			csd_lock(csd);
			if (wait)
				csd->node.u_flags |= CSD_TYPE_SYNC;
			csd->func = func;
			csd->info = info;
#ifdef CONFIG_CSD_LOCK_WAIT_DEBUG
			csd->node.src = smp_processor_id();
			csd->node.dst = cpu;
#endif
			trace_csd_queue_cpu(cpu, _RET_IP_, func, csd);

			if (llist_add(&csd->node.llist, &per_cpu(call_single_queue, cpu))) {
				__cpumask_set_cpu(cpu, cfd->cpumask_ipi);
				nr_cpus++;
				last_cpu = cpu;
			}
		}

		/*
		 * Choose the most efficient way to send an IPI. Note that the
		 * number of CPUs might be zero due to concurrent changes to the
		 * provided mask.
		 */
		if (nr_cpus == 1)
			send_call_function_single_ipi(last_cpu);
		else if (likely(nr_cpus > 1))
			send_call_function_ipi_mask(cfd->cpumask_ipi);
	}

	if (run_local && (!cond_func || cond_func(this_cpu, info))) {
		unsigned long flags;

		local_irq_save(flags);
		csd_do_func(func, info, NULL);
		local_irq_restore(flags);
	}

	if (run_remote && wait) {
		for_each_cpu(cpu, cfd->cpumask) {
			call_single_data_t *csd;

			csd = per_cpu_ptr(cfd->csd, cpu);
			csd_lock_wait(csd);
		}
	}
}
```
- 
