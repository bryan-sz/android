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

# cgroup文件系统配置
  上面我们看到了系统已经mount的cgroup文件系统，那么这些mount的文件系统是在哪里配置的呢？
  ack源码的system/core/libprocessgroup/profiles/cgroups.json配置文件，就是配置了系统的cgroup子系统，参考如下：
```
{
  "Cgroups": [
    {
      "Controller": "blkio",
      "Path": "/dev/blkio",
      "Mode": "0775",
      "UID": "system",
      "GID": "system"
    },
    {
      "Controller": "cpu",
      "Path": "/dev/cpuctl",
      "Mode": "0755",
      "UID": "system",
      "GID": "system"
    },
    {
      "Controller": "cpuset",
      "Path": "/dev/cpuset",
      "Mode": "0755",
      "UID": "system",
      "GID": "system"
    },
    {
      "Controller": "memory",
      "Path": "/dev/memcg",
      "Mode": "0700",
      "UID": "root",
      "GID": "system",
      "Optional": true
    }
  ],
  "Cgroups2": {
    "Path": "/sys/fs/cgroup",
    "Mode": "0775",
    "UID": "system",
    "GID": "system",
    "Controllers": [
      {
        "Controller": "freezer",
        "Path": "."
      }
    ]
  }
}

```

# cgroup初始化
cgroup初始化相当早，在[system/core/init/README.md](https://cs.android.com/android/platform/superproject/main/+/main:system/core/init/README.md;l=485?q=early-init)文档中就有这样的描述：
```
Trigger Sequence
----------------

Init uses the following sequence of triggers during early boot. These are the
built-in triggers defined in init.cpp.

   1. `early-init` - The first in the sequence, triggered after cgroups has been configured
      but before ueventd's coldboot is complete.
   2. `init` - Triggered after coldboot is complete.
   3. `charger` - Triggered if `ro.bootmode == "charger"`.
   4. `late-init` - Triggered if `ro.bootmode != "charger"`, or via healthd triggering a boot
      from charging mode.
```
由此可见，cgroup的初始化比early-init阶段还要早，肯定在ueventd启动之前；
事实上，在[system/core/init/init.cpp](https://cs.android.com/android/platform/superproject/main/+/main:system/core/init/init.cpp)的SecondStageMain函数中，进行了cgroup的初始化，大致顺序如下：
```
    am.QueueBuiltinAction(SetupCgroupsAction, "SetupCgroups");
    am.QueueBuiltinAction(SetKptrRestrictAction, "SetKptrRestrict");
    am.QueueBuiltinAction(TestPerfEventSelinuxAction, "TestPerfEventSelinux");
    am.QueueEventTrigger("early-init");
    am.QueueBuiltinAction(ConnectEarlyStageSnapuserdAction, "ConnectEarlyStageSnapuserd");
```
可以看到，SetupCgroupsAction确实比early-init要早一些。
## SetupCgroupsAction
```
static Result<void> SetupCgroupsAction(const BuiltinArguments&) {
    if (!CgroupsAvailable()) {
        LOG(INFO) << "Cgroups support in kernel is not enabled";
        return {};
    }
    // Have to create <CGROUPS_RC_DIR> using make_dir function
    // for appropriate sepolicy to be set for it
    make_dir(android::base::Dirname(CGROUPS_RC_PATH), 0711);
    if (!CgroupSetup()) {
        return ErrnoError() << "Failed to setup cgroups";
    }

    return {};
}
```
- 首先检查系统cgroup功能是否可用，CgroupsAvailable函数就是直接访问/proc/cgroups文件，确定kernel是否提供了cgroups功能；
- 创建/dev/cgroup_info/目录，从注释可以看到，是为了设置合理的sepolicy；
- 最重要的是，调用CgroupSetup真正的初始化cgroup了；
## SetupCgroupsAction
SetupCgroupsAction函数位于system/core/libprocessgroup/setup/cgroup_map_write.cpp文件中，注意，已经不是在init.cpp文件了，是专门的libprocessgroup目录。

- 参考文档：[Android中关于cpu/cpuset/schedtune的应用](https://www.cnblogs.com/arnoldlu/p/6221608.html)
