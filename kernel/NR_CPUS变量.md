# 什么是NR_CPUS
    简单来说，我们的系统都有支持的cpu数量的上限，这个也很好理解，毕竟很多资源都是预分配，而且预分配的和CPU数量有关系，那就只能定义一个系统能够支持的CPU的数量上限，这样就能避免浪费资源，同时也不用担心资源不够的问题；
    NR_CPUS变量就是承担了这样的责任，告诉系统中可以最多可以支持多少CPU；
# NR_CPUS配置
    Linux系统支持的场景千变万化，对CPU数量的支持也不同，如果直接定死，也不符合系统弹性要求。但是总要定下来啊，那么NR_CPUS到底定多少合适呢？就可以用到Kconfig机制了，毕竟构建镜像总是知道自己的使用场景的。以arm64为例，在arch/arm64/Kconfig中有如下配置：
```
config NR_CPUS
	int "Maximum number of CPUs (2-4096)"
	range 2 4096
	default "256"
```
这样，就提供了一个机制，让我们可以为自己的系统配置，究竟支持多少CPU；
# NR_CPUS变量定义
  上面在Kconfig中的配置，默认会生成CONFIG_NR_CPUS,那么NR_CPUS呢？在include/linux/threads.h头文件中，有如下定义：
```
/*
 * The default limit for the nr of threads is now in
 * /proc/sys/kernel/threads-max.
 */

/*
 * Maximum supported processors.  Setting this smaller saves quite a
 * bit of memory.  Use nr_cpu_ids instead of this except for static bitmaps.
 */
#ifndef CONFIG_NR_CPUS
/* FIXME: This should be fixed in the arch's Kconfig */
#define CONFIG_NR_CPUS	1
#endif

/* Places which use this should consider cpumask_var_t. */
#define NR_CPUS		CONFIG_NR_CPUS
```
  可以看到，NR_CPUS就是CONFIG_NR_CPUS宏，如果在make menuconfig的时候没有配置CONFIG_NR_CPUS，那么默认就是1；
  同时，也可以从注释中看到，将CONFIG_NR_CPUS尽可能设置的小一些，能够节约很多内存。
