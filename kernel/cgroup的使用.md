# cgroup在android中的使用
  在Android中，cgroup发挥了重要的作用，目前只了解到划分了top-app,foreground,background几个分组，用于设置不同的调度策略和参数。
  要使用cgroup，首先必须mount cgroup目录，以手上的一台Android平板为例，mount情况如下：
```
TB328XU:/ $ mount | grep cgroup
none on /dev/blkio type cgroup (rw,nosuid,nodev,noexec,relatime,blkio)
none on /dev/cg2_bpf type cgroup2 (rw,nosuid,nodev,noexec,relatime)
none on /dev/cpuctl type cgroup (rw,nosuid,nodev,noexec,relatime,cpu)
none on /acct type cgroup (rw,nosuid,nodev,noexec,relatime,cpuacct)
none on /dev/cpuset type cgroup (rw,nosuid,nodev,noexec,relatime,cpuset,noprefix,release_agent=/sbin/cpuset_release_agent)
none on /dev/freezer type cgroup (rw,nosuid,nodev,noexec,relatime,freezer)
none on /dev/memcg type cgroup (rw,nosuid,nodev,noexec,relatime,memory)
none on /dev/stune type cgroup (rw,nosuid,nodev,noexec,relatime,schedtune)
```
注意其中的参数，cpuctl和cpuset的mount参数一个是cpu子系统，一个是cpuset子系统；另外还有/dev/stune是schedtune；




- 参考文档：(Android中关于cpu/cpuset/schedtune的应用)[https://www.cnblogs.com/arnoldlu/p/6221608.html]
