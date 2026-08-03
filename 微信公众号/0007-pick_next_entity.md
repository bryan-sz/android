# 1. pick_next_entity函数解读
- 本文档以linux内核6.15-rc3作为源码基线。
- 在[pick_next_task_fair函数解读](https://mp.weixin.qq.com/s/EWfg1DrmMX-j8iP-XUmiIA)中讲到`pick_task_fair`函数会调用到`pick_next_entity`函数得到对应的调度实体，本篇文章就针对`pick_next_entity`进行解读

# 2. 函数定义
- `pick_next_entity`函数定义在kernel/sched/fair.c文件第5583行，完整定义如下：
```
/*
 * Pick the next process, keeping these things in mind, in this order:
 * 1) keep things fair between processes/task groups
 * 2) pick the "next" process, since someone really wants that to run
 * 3) pick the "last" process, for cache locality
 * 4) do not run the "skip" process, if something else is available
 */
static struct sched_entity *
pick_next_entity(struct rq *rq, struct cfs_rq *cfs_rq)
{
	struct sched_entity *se;

	/*
	 * Picking the ->next buddy will affect latency but not fairness.
	 */
	if (sched_feat(PICK_BUDDY) &&
	    cfs_rq->next && entity_eligible(cfs_rq, cfs_rq->next)) {
		/* ->next will never be delayed */
		WARN_ON_ONCE(cfs_rq->next->sched_delayed);
		return cfs_rq->next;
	}

	se = pick_eevdf(cfs_rq);
	if (se->sched_delayed) {
		dequeue_entities(rq, se, DEQUEUE_SLEEP | DEQUEUE_DELAYED);
		/*
		 * Must not reference @se again, see __block_task().
		 */
		return NULL;
	}
	return se;
}
```
## 2.1. 注释说明
- 这个函数就是挑选出下一个可以运行的任务，需要记住如下关键事项，优先顺序如下：
    1. 在进程和进程组之间要保持公平
    2. 尽可能的挑选出下一个任务
    3. 如果没有下一个任务，从cache的局部性触发，还是挑选出上一个任务
    4. 如果有可用任务的话，不要运行标注skip的任务
## 2.2. 返回值说明
- 之前的pick_next系列函数操作的对应都是`rq`和`struct task_struct`，这个函数开始引入`struct sched_entity`，对应的就是调度实体，并且返回值也是`sched_entity`，就是挑选的真正的调度实体
  
## 2.3. 函数流程
- 函数很简单，首先判断`PICK_BUDDY`这个entity是否打开，这个我们暂时不关注，可以先跳过；
- 然后就是通过`pick_eevdf`从红黑树上挑选出来第一个合适的调度实体
- `pick_eevdf`就是从linux-6.6开始引入内核的eevdf算法的实现，虽然借用了cfs的框架，但是内部算法已经不再是完全根据vruntime计算的公平性了，并最终返回对应的se；这个就留到后续详细展开
- `se->sched_delayed`这个判断，表明挑选出来的函数是一个标记了延迟出队的任务，因此需要调用`dequeue_entities`将其出队，而且因为不是一个合适的调度实体，因此最终返回NULL

# 3. 核心调用链
- pick_next_entity
  - pick_eevdf
  - se->sched_delayed
    - dequeue_entity

# 4. 总结
- `pick_next_entity`函数，从前面的`struct task_struct`转到了真正的调度实体`struct sched_entity`，起到了承上启下的作用
