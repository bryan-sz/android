# pick_next_task函数解读
- 本文档以linux内核6.15-rc3作为源码基线。

## 函数定义
- pick_next_task函数定义在kernel/sched/core.c文件有两个定义，分别位于6105行和6549行，其中6105行的是开启了CONFIG_SCHED_CORE这个宏定义的基础上的定义，这个主要是服务于有SMT功能的CPU的，因此暂时不考虑。
- 常规的pick_next_task函数实现如下：
```
static struct task_struct *
pick_next_task(struct rq *rq, struct task_struct *prev, struct rq_flags *rf)
{
	return __pick_next_task(rq, prev, rf);
}
```
- 这个函数很简单，其实就是一个包装函数，转头就调用了__pick_next_task函数，这个才是真正的实现。
- 至于为什么要这么包装一次，可以参考这次commit：[fd03c5g](https://link.wtturl.cn/?target=https%3A%2F%2Fgit.kernel.org%2Ftip%2Ffd03c5b8585562d60f8b597b4332d28f48abfe7d&scene=im&aid=497858&lang=zh)
- 简单说，这是在原有调度器核心的一次简单的重构，主要实现以下目的：
    1.  旧逻辑调度类的`pick_next_task` = `pick_task` + `set_next_task(first=true)`，而各个调度类都有自己的实现，所以需要统一；
    2.  新逻辑里面，全局统一流程：`pick_next_task(prev)` = `pick_task` + `put_prev_task(prev)` + `set_next_task`
    3.  核心动机其实是为了`sched_ext`（BPF 调度器扩展）提供统一回调机制，因为如果调度器的语义不一致，而且事实上各个调度器也都有不同的实现，那么sched_ext就没办法覆盖所有调度类的任务进行扩展。因此，为了后续的sched_ext准备，必须将语义归一；
    4.  作为一个附加的收益，将各个调度器类语义进行了归一，也是对调度器核心代码的一次清理。

## __pick_next_task
- __pick_next_task函数定义在kernel/sched/core.c文件的6003行，定义如下：
```
/*
 * Pick up the highest-prio task:
 */
static inline struct task_struct *
__pick_next_task(struct rq *rq, struct task_struct *prev, struct rq_flags *rf)
{
	const struct sched_class *class;
	struct task_struct *p;

	rq->dl_server = NULL;

	if (scx_enabled())
		goto restart;

	/*
	 * Optimization: we know that if all tasks are in the fair class we can
	 * call that function directly, but only if the @prev task wasn't of a
	 * higher scheduling class, because otherwise those lose the
	 * opportunity to pull in more work from other CPUs.
	 */
	if (likely(!sched_class_above(prev->sched_class, &fair_sched_class) &&
		   rq->nr_running == rq->cfs.h_nr_queued)) {

		p = pick_next_task_fair(rq, prev, rf);
		if (unlikely(p == RETRY_TASK))
			goto restart;

		/* Assume the next prioritized class is idle_sched_class */
		if (!p) {
			p = pick_task_idle(rq);
			put_prev_set_next_task(rq, prev, p);
		}

		return p;
	}

restart:
	prev_balance(rq, prev, rf);

	for_each_active_class(class) {
		if (class->pick_next_task) {
			p = class->pick_next_task(rq, prev);
			if (p)
				return p;
		} else {
			p = class->pick_task(rq);
			if (p) {
				put_prev_set_next_task(rq, prev, p);
				return p;
			}
		}
	}

	BUG(); /* The idle class should always have a runnable task. */
}
```
### 注释解读
- __pick_next_task函数只有一行注释，直截了当的说明了函数的作用：挑选出来最高优先级的任务

### sched_ext检查
```
	if (scx_enabled())
		goto restart;
```
- 首先判断是否开启了sched_ext并加载了对应的调度类，那么就直接走到restart，兼容sched_ext调度类的函数逻辑。

### 快速路径处理
```
    if (likely(!sched_class_above(prev->sched_class, &fair_sched_class) && rq->nr_running == rq->cfs.h_nr_queued)) {
    		p = pick_next_task_fair(rq, prev, rf);
		if (unlikely(p == RETRY_TASK))
			goto restart;

		/* Assume the next prioritized class is idle_sched_class */
		if (!p) {
			p = pick_task_idle(rq);
			put_prev_set_next_task(rq, prev, p);
		}

		return p;
	}
```
- 在对应的注释中说明了如下要求：如果prev不是比CFS调度类还高的任务，就可以考虑直接调用CFS调度类的pick_next_task实现了，这里有一个前提，就是系统中肯定绝大部分任务都是CFS调度类，因此直接走CFS调度类的实现是一个快速路径；
- 如果prev是比CFS还要高的调度类任务，那就不能走这个快速路径。毕竟有可能还有其他的比CFS还要高的调度类任务等待执行，这就刚好是一个时机，不能错过。
- 比较prev任务调度类和CFS调度类优先级，以及当前rq中是否全部都是CFS任务，如果这两个条件同时满足，就直接调用CFS调度类的pick_next_task_fair任务。挑选到就直接返回。如果挑选过程出错，就跳转到标记为restart的正常路径；
- 如果pick_next_task_fair没有挑选到任务，那就使用pick_task_idle，运行idle任务；
- 需要注意的是，pick_task_task_fair返回结果，一个是RETRY_TASK，一个是NULL，这是不同的。

### 正常路径处理
```
restart:
	prev_balance(rq, prev, rf);

	for_each_active_class(class) {
		if (class->pick_next_task) {
			p = class->pick_next_task(rq, prev);
			if (p)
				return p;
		} else {
			p = class->pick_task(rq);
			if (p) {
				put_prev_set_next_task(rq, prev, p);
				return p;
			}
		}
	}

	BUG(); /* The idle class should always have a runnable task. */
}
```
- 首先，抓住机会，进行一次load balance
- 其次，遍历系统中的调度类，根据系统中的调度类的`pick_next_task`函数钩子实现情况，决定调用对应调度类的`pick_next_task`还是`pick_task` + `put_prev_set_next_task`的组合。
- 至于真正的`pick_next_task`的函数实现逻辑，就放到各个调度类去具体实现。这样，就实现了调度器核心与多个调度器类的解耦，以及各个调度类的多样性。
- 需要说明的是，如果走到最后的`BUG()`这里，就说明系统异常了，因为如果没有合适的任务，需要运行到IDLE调度类的idle任务。如果idle任务都没有被选出来，那系统就是出现问题了。因此，主动调用`BUG()`将问题暴露出来。

## 核心调用链
根据以上解读，pick_next_task的核心调用链如下：
- pick_next_task
  - __pick_next_task
  - scx_enabled
  - prev_balance
  - for_each_active_class
    - sched_class -> pick_next_task

如果是快速路径，核心调用链计划位直接调用CFS调度类的实现：
- pick_next_task
  - __pick_next_task
    - scx_enabled
    - pick_next_task_fair (CFS调度类的pick_next_task实现)
    - pick_task_idle (IDLE调度类的pick_next_task实现)

## 总结
- pick_next_task是调度器的核心函数，主要是挑选出下一个运行的最高优先级的任务
- 如果可能，进行一次load balance
- Linux内核中存在多个调度器类，包含Deadline、RT、CFS、EXT、ILDE等，并调用对应调度类的具体实现
- 如果没有可运行任务，就挑选idle任务进入IDLE状态
- 针对系统中99%以上是CFS调度类的特点，进行了快速路径的优化，这在争分夺秒的调度时延中显得很重要。
