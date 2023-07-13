# Android启动
- 要搞清楚Android系统，从Android的启动来分析，是一个不错的角度，也是我之前学习Linux系统的一个心得；
- 众所周知，Android是基于Linux内核开发的操作系统，而我之前一直从事的是嵌入式系统的Linux内核开发，所以对于我来说，从bootloader初始化，引导Linux内核执行最终跳转到init这个用户态第一个进程，这个流程是熟悉的不能再熟悉，不同的是Android上的init进程与之前的嵌入式系统上busybox的init有变化，所以就从init这个阶段开始研究；
- init进程引导，最终Android设备进入到稳态能够让用户操作设备，这就是这一篇启动文章要研究的范围；
- 所有的代码参考的是Android 13代码

## 启动过程
在system/core/init目录下，有main.cpp文件中，有init进程的main函数入口：
```
int main(int argc, char** argv) {
#if __has_feature(address_sanitizer)
    __asan_set_error_report_callback(AsanReportCallback);
#elif __has_feature(hwaddress_sanitizer)
    __hwasan_set_error_report_callback(AsanReportCallback);
#endif
    // Boost prio which will be restored later
    setpriority(PRIO_PROCESS, 0, -20);
    if (!strcmp(basename(argv[0]), "ueventd")) {
        return ueventd_main(argc, argv);
    }

    if (argc > 1) {
        if (!strcmp(argv[1], "subcontext")) {
            android::base::InitLogging(argv, &android::base::KernelLogger);
            const BuiltinFunctionMap& function_map = GetBuiltinFunctionMap();

            return SubcontextMain(argc, argv, &function_map);
        }

        if (!strcmp(argv[1], "selinux_setup")) {
            return SetupSelinux(argv);
        }

        if (!strcmp(argv[1], "second_stage")) {
            return SecondStageMain(argc, argv);
        }
    }

    return FirstStageMain(argc, argv);
}
```
代码非常简单，分为如下几个点：
- 如果打开了asan、hwasan的话，调用asan、hwasan的回调函数；
- 通过setpriority设置init进程自身的优先级值为-20，保证是CFS进程的最高优先级；
- 如果main.cpp文件参与了ueventd的编译并且是ueventd启动的，那就跳转到ueventd_main函数入口；
- 如果传递了subcontext、selinux_setup或者second_stage启动参数，就跳转到对应的启动阶段；
- 默认跳转到FirstStageMain函数入口；
通过以上分析，起码init进程会分成两个阶段启动，即FirstStageMain和SecondStageMain两个入口；

### 第一阶段
FirstStageMain函数定义在system/core/init目录下的first_stage_init.cpp文件中，函数功能较复杂，下面分段进行解读：
#### 复位信号处理
```
int FirstStageMain(int argc, char** argv) {
    if (REBOOT_BOOTLOADER_ON_PANIC) {
        InstallRebootSignalHandlers();
    }
```
- 首先我也不知道REBOOT_BOOTLOADER_ON_PANIC是什么，但是看起来像一个宏定义开关，通过名字推测应该是在init进程崩溃或者内核panic的时候要做的一些处理；
- InstallRebootSignalHandlers函数名字很清楚，就是install一些reboot的信号处理函数；
- InstallRebootSignalHandlers函数在system/core/init/reboot_utils.cpp文件中定义，可以详细看一下：
```
void InstallRebootSignalHandlers() {
    // Instead of panic'ing the kernel as is the default behavior when init crashes,
    // we prefer to reboot to bootloader on development builds, as this will prevent
    // boot looping bad configurations and allow both developers and test farms to easily
    // recover.
    struct sigaction action;
    memset(&action, 0, sizeof(action));
    sigfillset(&action.sa_mask);
    action.sa_handler = [](int signal) {
        // These signal handlers are also caught for processes forked from init, however we do not
        // want them to trigger reboot, so we directly call _exit() for children processes here.
        if (getpid() != 1) {
            _exit(signal);
        }

        // Calling DoReboot() or LOG(FATAL) is not a good option as this is a signal handler.
        // RebootSystem uses syscall() which isn't actually async-signal-safe, but our only option
        // and probably good enough given this is already an error case and only enabled for
        // development builds.
        InitFatalReboot(signal);
    };
    action.sa_flags = SA_RESTART;
    sigaction(SIGABRT, &action, nullptr);
    sigaction(SIGBUS, &action, nullptr);
    sigaction(SIGFPE, &action, nullptr);
    sigaction(SIGILL, &action, nullptr);
    sigaction(SIGSEGV, &action, nullptr);
#if defined(SIGSTKFLT)
    sigaction(SIGSTKFLT, &action, nullptr);
#endif
    sigaction(SIGSYS, &action, nullptr);
    sigaction(SIGTRAP, &action, nullptr);
}
```
- 整体来说，就是给SIGABRT/SIGBUS/SIGFPE/SIGILL/SIGSEGV/SIGSYS/SIGTRAP以及SIGSTKFLT（打开SIGSTKFLT开关）统一安装了信号处理函数；
- 信号处理函数逻辑很简单，如果不是init，直接_exit退出进程就好；反之如果是进程（进程号是1），就执行InitFatalReboot函数，将设备进行reboot；
- 那么InitFatalReboot函数是怎么reboot复位设备的呢？
```
void __attribute__((noreturn)) InitFatalReboot(int signal_number) {
    auto pid = fork();

    if (pid == -1) {
        // Couldn't fork, don't even try to backtrace, just reboot.
        RebootSystem(ANDROID_RB_RESTART2, init_fatal_reboot_target);
    } else if (pid == 0) {
        // Fork a child for safety, since we always want to shut down if something goes wrong, but
        // its worth trying to get the backtrace, even in the signal handler, since typically it
        // does work despite not being async-signal-safe.
        sleep(5);
        RebootSystem(ANDROID_RB_RESTART2, init_fatal_reboot_target);
    }

    // In the parent, let's try to get a backtrace then shutdown.
    LOG(ERROR) << __FUNCTION__ << ": signal " << signal_number;
    unwindstack::AndroidLocalUnwinder unwinder;
    unwindstack::AndroidUnwinderData data;
    if (!unwinder.Unwind(data)) {
        LOG(ERROR) << __FUNCTION__ << ": Failed to unwind callstack: " << data.GetErrorString();
    }
    for (const auto& frame : data.frames) {
        LOG(ERROR) << unwinder.FormatFrame(frame);
    }
    if (init_fatal_panic) {
        LOG(ERROR) << __FUNCTION__ << ": Trigger crash";
        android::base::WriteStringToFile("c", PROC_SYSRQ);
        LOG(ERROR) << __FUNCTION__ << ": Sys-Rq failed to crash the system; fallback to exit().";
        _exit(signal_number);
    }
    RebootSystem(ANDROID_RB_RESTART2, init_fatal_reboot_target);
}
```
- 首先通过fork调用，派生一个子进程；
- 如果fork调用时便，直接调用RebootSystem复位；
- 如果fork派生子进程成功，先sleep 5秒再调用RebootSystem复位，为什么会sleep5秒呢，通过注释来看，主要是想让init进程能够有足够的时间收集进程崩溃的backtrace；
- init进程首先记录导致reboot的信号；
- 尝试使用unwinder进行推栈；
- 如果打开了init_fatal_panic配置开关，直接通过对/proc/sysrq-trigger文件写c造内核crash从而重启系统；
- 各种措施执行之后如果还没有复位，init再尝试调用RebootSystem进行复位，至于RebootSystem实现，从入参格式来看，如果熟悉linux c语言的话，基本就是调用reboot系统调用了；

至此，给init进程安装复位信号处理函数的操作就分析清楚；
#### 启动时间戳
代码很简单，就只有下面这一句，感觉是获取当前时间戳的一个方法，至于boot_clock的实现方法，暂时先忽略；
```
boot_clock::time_point start_time = boot_clock::now();
```
#### error日志机制
```
    std::vector<std::pair<std::string, int>> errors;
#define CHECKCALL(x) \
    if ((x) != 0) errors.emplace_back(#x " failed", errno);
```
- 首先定义了一个errors的vector变量，用于记录error错误；
- CHECKCALL宏应该是一个老司机的手法，融合了多个c语言的宏技巧；首先是根据x表达式的执行结果来判断是否记录错误信息，其次#连接执行异常的表达式，最后记录errno；
- 另外提一句，根据提交记录，这个老司机也一样掉了宏定义中经常掉的坑，可以详细参考：[修复clang-tidy告警](https://android-review.googlesource.com/c/platform/system/core/+/1012395/4/init/first_stage_init.cpp#155)
#### 执行环境准备
```
// Clear the umask.
    umask(0);
    CHECKCALL(clearenv());
    CHECKCALL(setenv("PATH", _PATH_DEFPATH, 1));
```
- 首先将umask清空；
- 再将环境变量清空；
- 设置PATH环境变量，其中_PATH_DEFPATH定义在bionic/libc/include/path.h头文件中，定义如下：
```
/** Default shell search path. */
#define _PATH_DEFPATH "/product/bin:/apex/com.android.runtime/bin:/apex/com.android.art/bin:/system_ext/bin:/system/bin:/system/xbin:/odm/bin:/vendor/bin:/vendor/xbin"
```
#### initramfs装载
```
// Get the basic filesystem setup we need put together in the initramdisk
    // on / and then we'll let the rc file figure out the rest.
    CHECKCALL(mount("tmpfs", "/dev", "tmpfs", MS_NOSUID, "mode=0755"));
    CHECKCALL(mkdir("/dev/pts", 0755));
    CHECKCALL(mkdir("/dev/socket", 0755));
    CHECKCALL(mkdir("/dev/dm-user", 0755));
    CHECKCALL(mount("devpts", "/dev/pts", "devpts", 0, NULL));
#define MAKE_STR(x) __STRING(x)
    CHECKCALL(mount("proc", "/proc", "proc", 0, "hidepid=2,gid=" MAKE_STR(AID_READPROC)));
#undef MAKE_STR
    // Don't expose the raw commandline to unprivileged processes.
    CHECKCALL(chmod("/proc/cmdline", 0440));
    std::string cmdline;
    android::base::ReadFileToString("/proc/cmdline", &cmdline);
    // Don't expose the raw bootconfig to unprivileged processes.
    chmod("/proc/bootconfig", 0440);
    std::string bootconfig;
    android::base::ReadFileToString("/proc/bootconfig", &bootconfig);
    gid_t groups[] = {AID_READPROC};
    CHECKCALL(setgroups(arraysize(groups), groups));
    CHECKCALL(mount("sysfs", "/sys", "sysfs", 0, NULL));
    CHECKCALL(mount("selinuxfs", "/sys/fs/selinux", "selinuxfs", 0, NULL));

    CHECKCALL(mknod("/dev/kmsg", S_IFCHR | 0600, makedev(1, 11)));

    if constexpr (WORLD_WRITABLE_KMSG) {
        CHECKCALL(mknod("/dev/kmsg_debug", S_IFCHR | 0622, makedev(1, 11)));
    }

    CHECKCALL(mknod("/dev/random", S_IFCHR | 0666, makedev(1, 8)));
    CHECKCALL(mknod("/dev/urandom", S_IFCHR | 0666, makedev(1, 9)));

    // This is needed for log wrapper, which gets called before ueventd runs.
    CHECKCALL(mknod("/dev/ptmx", S_IFCHR | 0666, makedev(5, 2)));
    CHECKCALL(mknod("/dev/null", S_IFCHR | 0666, makedev(1, 3)));

    // These below mounts are done in first stage init so that first stage mount can mount
    // subdirectories of /mnt/{vendor,product}/.  Other mounts, not required by first stage mount,
    // should be done in rc files.
    // Mount staging areas for devices managed by vold
    // See storage config details at http://source.android.com/devices/storage/
    CHECKCALL(mount("tmpfs", "/mnt", "tmpfs", MS_NOEXEC | MS_NOSUID | MS_NODEV,
                    "mode=0755,uid=0,gid=1000"));
    // /mnt/vendor is used to mount vendor-specific partitions that can not be
    // part of the vendor partition, e.g. because they are mounted read-write.
    CHECKCALL(mkdir("/mnt/vendor", 0755));
    // /mnt/product is used to mount product-specific partitions that can not be
    // part of the product partition, e.g. because they are mounted read-write.
    CHECKCALL(mkdir("/mnt/product", 0755));

    // /debug_ramdisk is used to preserve additional files from the debug ramdisk
    CHECKCALL(mount("tmpfs", "/debug_ramdisk", "tmpfs", MS_NOEXEC | MS_NOSUID | MS_NODEV,
                    "mode=0755,uid=0,gid=0"));

    // /second_stage_resources is used to preserve files from first to second
    // stage init
    CHECKCALL(mount("tmpfs", kSecondStageRes, "tmpfs", MS_NOEXEC | MS_NOSUID | MS_NODEV,
                    "mode=0755,uid=0,gid=0"))
#undef CHECKCALL
```
- 这一段代码有点多，不过基本上就是如下几个事情：mount文件系统，mknod创建节点；
- 值得注意的是，中间从/proc/cmdline和/proc/bootconfig读取到了文件中；
- 读取/proc文件需要权限控制，这里的权限就是只有root和gid是AID_READPROC的用户能够查看proc接口；
- AID_READPROC定义在system/core/libcutils/include/private/android_filesystem_config.h文件中；
`#define AID_READPROC 3009     /* Allow /proc read access */`
- CHECKALL宏定义使用场景结束，将宏undef掉；
#### 日志初始化
```
    SetStdioToDevNull(argv);
    // Now that tmpfs is mounted on /dev and we have /dev/kmsg, we can actually
    // talk to the outside world...
    InitKernelLogging(argv);

    if (!errors.empty()) {
        for (const auto& [error_string, error_errno] : errors) {
            LOG(ERROR) << error_string << " " << strerror(error_errno);
        }
        LOG(FATAL) << "Init encountered errors starting first stage, aborting";
    }

    LOG(INFO) << "init first stage started!";
```
- 首先SetStdioToDevNull将stdin、stdout、stderr重定向到/dev/null；
- InitKernelLogging初始化日志；
- 检测上一步initramfs装载过程中的错误信息，如果有错误信息调用LOG(FATAL)直接abort退出进程；这一要注意，FATAL级别会导致LOG调用abort；
- 反之，则记录启动成功日志；
#### 检测initramfs
通过访问“/”根目录，确认initramfs是否挂载成功；
```
   auto old_root_dir = std::unique_ptr<DIR, decltype(&closedir)>{opendir("/"), closedir};
    if (!old_root_dir) {
        PLOG(ERROR) << "Could not opendir(\"/\"), not freeing ramdisk";
    }

    struct stat old_root_info;
    if (stat("/", &old_root_info) != 0) {
        PLOG(ERROR) << "Could not stat(\"/\"), not freeing ramdisk";
        old_root_dir.reset();
    }
```
- 感觉是一个lamda表达式，去打开和关闭“/”根目录，并将结果保存到old_root_dir；
- 通过stat访问“/”目录；这个感觉和上一步opendir&closedir有重复；
#### 加载内核模块
```
    auto want_console = ALLOW_FIRST_STAGE_CONSOLE ? FirstStageConsole(cmdline, bootconfig) : 0;
    auto want_parallel =
            bootconfig.find("androidboot.load_modules_parallel = \"true\"") != std::string::npos;

    boot_clock::time_point module_start_time = boot_clock::now();
    int module_count = 0;
    BootMode boot_mode = GetBootMode(cmdline, bootconfig);
    if (!LoadKernelModules(boot_mode, want_console,
                           want_parallel, module_count)) {
        if (want_console != FirstStageConsoleParam::DISABLED) {
            LOG(ERROR) << "Failed to load kernel modules, starting console";
        } else {
            LOG(FATAL) << "Failed to load kernel modules";
        }
    }
    if (module_count > 0) {
        auto module_elapse_time = std::chrono::duration_cast<std::chrono::milliseconds>(
                boot_clock::now() - module_start_time);
        setenv(kEnvInitModuleDurationMs, std::to_string(module_elapse_time.count()).c_str(), 1);
        LOG(INFO) << "Loaded " << module_count << " kernel modules took "
                  << module_elapse_time.count() << " ms";
    }

    bool created_devices = false;
    if (want_console == FirstStageConsoleParam::CONSOLE_ON_FAILURE) {
        if (!IsRecoveryMode()) {
            created_devices = DoCreateDevices();
            if (!created_devices) {
                LOG(ERROR) << "Failed to create device nodes early";
            }
        }
        StartConsole(cmdline);
    }
```
- 首先根据配置，获取console/是否并行加载内核模块配置；
- 调用system/core/libmodprobe/libmodprobe.cpp文件定义的Modprobe类中的方法，最终采用insmod或者modprobe加载内核模块；
- 统计加载内核模块的数量和时间；
- 判断是否recovery模式，如果非recovery模式，还要创建逻辑分区；
#### prop文件准备
```
    if (access(kBootImageRamdiskProp, F_OK) == 0) {
        std::string dest = GetRamdiskPropForSecondStage();
        std::string dir = android::base::Dirname(dest);
        std::error_code ec;
        if (!fs::create_directories(dir, ec) && !!ec) {
            LOG(FATAL) << "Can't mkdir " << dir << ": " << ec.message();
        }
        if (!fs::copy_file(kBootImageRamdiskProp, dest, ec)) {
            LOG(FATAL) << "Can't copy " << kBootImageRamdiskProp << " to " << dest << ": "
                       << ec.message();
        }
        LOG(INFO) << "Copied ramdisk prop to " << dest;
    }
```
- kBootImageRamdiskProp变量在system/core/init/second_stage_resources.h文件中定义；
```
constexpr const char kSecondStageRes[] = "/second_stage_resources";
constexpr const char kBootImageRamdiskProp[] = "/system/etc/ramdisk/build.prop";

inline std::string GetRamdiskPropForSecondStage() {
    return std::string(kSecondStageRes) + kBootImageRamdiskProp;
}
```
- 创建/second_stage_resources//system/etc/ramdisk/路径；
- 将/system/etc/ramdisk/build.prop文件copy到新创建的/second_stage_resources//system/etc/ramdisk/路径下；
- build.prop该文件包含构建的时间戳信息，移动到/second_stage_resources以便第二阶段init可以读取启动的构建时间戳；
#### debug模式
```
    // If "/force_debuggable" is present, the second-stage init will use a userdebug
    // sepolicy and load adb_debug.prop to allow adb root, if the device is unlocked.
    if (access("/force_debuggable", F_OK) == 0) {
        constexpr const char adb_debug_prop_src[] = "/adb_debug.prop";
        constexpr const char userdebug_plat_sepolicy_cil_src[] = "/userdebug_plat_sepolicy.cil";
        std::error_code ec;  // to invoke the overloaded copy_file() that won't throw.
        if (access(adb_debug_prop_src, F_OK) == 0 &&
            !fs::copy_file(adb_debug_prop_src, kDebugRamdiskProp, ec)) {
            LOG(WARNING) << "Can't copy " << adb_debug_prop_src << " to " << kDebugRamdiskProp
                         << ": " << ec.message();
        }
        if (access(userdebug_plat_sepolicy_cil_src, F_OK) == 0 &&
            !fs::copy_file(userdebug_plat_sepolicy_cil_src, kDebugRamdiskSEPolicy, ec)) {
            LOG(WARNING) << "Can't copy " << userdebug_plat_sepolicy_cil_src << " to "
                         << kDebugRamdiskSEPolicy << ": " << ec.message();
        }
        // setenv for second-stage init to read above kDebugRamdisk* files.
        setenv("INIT_FORCE_DEBUGGABLE", "true", 1);
    }
```
- 首先检测是否存在文件/force_debuggable；
- 如果存在/force_debuggable，再检测是否存在/adb_debug.prop和/userdebug_plat_sepolicy.cil文件，并将其copy到/debug_ramdisk目录下；
- 设置环境变量INIT_FORCE_DEBUGGABLE为true；
#### chroot
```
    if (ForceNormalBoot(cmdline, bootconfig)) {
        mkdir("/first_stage_ramdisk", 0755);
        PrepareSwitchRoot();
        // SwitchRoot() must be called with a mount point as the target, so we bind mount the
        // target directory to itself here.
        if (mount("/first_stage_ramdisk", "/first_stage_ramdisk", nullptr, MS_BIND, nullptr) != 0) {
            PLOG(FATAL) << "Could not bind mount /first_stage_ramdisk to itself";
        }
        SwitchRoot("/first_stage_ramdisk");
    }
    if (!DoFirstStageMount(!created_devices)) {
        LOG(FATAL) << "Failed to mount required partitions early ...";
    }

    struct stat new_root_info;
    if (stat("/", &new_root_info) != 0) {
        PLOG(ERROR) << "Could not stat(\"/\"), not freeing ramdisk";
        old_root_dir.reset();
    }

    if (old_root_dir && old_root_info.st_dev != new_root_info.st_dev) {
        FreeRamdisk(old_root_dir.get(), old_root_info.st_dev);
    }
```
- 在根目录下创建/first_stage_ramdisk目录；
- PrepareSwitchRoot做切根前的准备，主要是创建/first_stage_ramdisk/system/bin/snapuserd目录，并且将/system/bin/snapuserd或/system/bin/snapuserd_ramdisk链接过去；
- mount/first_stage_ramdisk目录，注意要切根，所以用的是bind选项的mount；
- SwitchRoot切根到/first_stage_ramdisk，最终调用的是chroot完成切根动作；
- 因为切了根，所以需要DoFirstStageMount再次完成相关设备和目录的mount；
- 调用FreeRamdisk反复释放initramfs；
#### avb校验
```
    SetInitAvbVersionInRecovery();

    setenv(kEnvFirstStageStartedAt, std::to_string(start_time.time_since_epoch().count()).c_str(),
           1);
```
- 第一行代码就是检查avb，并设置INIT_AVB_VERSION环境变量，关于avb，后续详细再了解；
- 第二行代码也是设置启动时间的环境变量；
#### 跳转到第二阶段init
```
const char* path = "/system/bin/init";
    const char* args[] = {path, "selinux_setup", nullptr};
    auto fd = open("/dev/kmsg", O_WRONLY | O_CLOEXEC);
    dup2(fd, STDOUT_FILENO);
    dup2(fd, STDERR_FILENO);
    close(fd);
    execv(path, const_cast<char**>(args));

    // execv() only returns if an error happened, in which case we
    // panic and never fall through this conditional.
    PLOG(FATAL) << "execv(\"" << path << "\") failed";

    return 1;
```
- 调用execve函数fork一个子进程，子进程执行/system/bin/init二进制，也就是第二阶段的init；
- 传递给init的参数包含selinux_setup，也就是可以猜测，android是默认强制启动了selinux的；
至此，第一阶段init初始化完成，后续分析下一阶段的init；
### selinux启动
参考main.cpp文件中，有init进程的main函数入口中，根据传入的参数进入不同的启动阶段，在第一阶段init传递的是selinux_setup参数，所以会调用SetupSelinux来启动selinux；
SetupSelinux定义在system/core/init/selinux.cpp文件中，下面就针对SetupSelinux来拆解selinux启动过程；
#### SetupSelinux注释
- 在SetupSelinux函数前有一段注释，将Selinux启动过程注意事项和流程讲解的很清楚，可以参考；
```
// The SELinux setup process is carefully orchestrated around snapuserd. Policy
// must be loaded off dynamic partitions, and during an OTA, those partitions
// cannot be read without snapuserd. But, with kernel-privileged snapuserd
// running, loading the policy will immediately trigger audits.
//
// We use a five-step process to address this:
//  (1) Read the policy into a string, with snapuserd running.
//  (2) Rewrite the snapshot device-mapper tables, to generate new dm-user
//      devices and to flush I/O.
//  (3) Kill snapuserd, which no longer has any dm-user devices to attach to.
//  (4) Load the sepolicy and issue critical restorecons in /dev, carefully
//      avoiding anything that would read from /system.
//  (5) Re-launch snapuserd and attach it to the dm-user devices from step (2).
//
// After this sequence, it is safe to enable enforcing mode and continue booting.
```
- 在这里感叹一句，这个注释生动的说明了什么样的注释才是一个好的注释，不是简单说明要怎么做，而是说明为什么要这么做；
- 注释里面提到的sapuserd，看名字应该是一个daemon进程，但是不知道具体功能是啥，也没有注意到在第一阶段什么时候启动的snapuserd，回头再补充；
- snapuserd和dm-user的详细功能，可以参考[虚拟A/B概述](https://source.android.google.cn/devices/tech/ota/virtual_ab?hl=zh-cn#dm-user)；
#### 复位信号处理
```
int SetupSelinux(char** argv) {
    SetStdioToDevNull(argv);
    InitKernelLogging(argv);

    if (REBOOT_BOOTLOADER_ON_PANIC) {
        InstallRebootSignalHandlers();
    }
```
- 和init第一阶段一样，先初始化日志，然后安装复位信号处理函数；
#### 启动时间戳
```
boot_clock::time_point start_time = boot_clock::now();
```
- 一行代码，获取当前时间；
- 其实可以考虑将这个时间戳放到复位信号处理前面，这样是更准确的SetupSelinux函数时间戳；
#### mount文件系统
```
MountMissingSystemPartitions();
```
- 一行代码，这个函数同样定义在system/core/init/selinux.cpp文件中，详细分析一下这个函数功能；
```
// This is for R system.img/system_ext.img to work on old vendor.img as system_ext.img
// is introduced in R. We mount system_ext in second stage init because the first-stage
// init in boot.img won't be updated in the system-only OTA scenario.
void MountMissingSystemPartitions() {
```
- 注释详细解释了是因为Android R的OTA场景而引入的；
```
    android::fs_mgr::Fstab fstab;
    if (!ReadDefaultFstab(&fstab)) {
        LOG(ERROR) << "Could not read default fstab";
    }

    android::fs_mgr::Fstab mounts;
    if (!ReadFstabFromFile("/proc/mounts", &mounts)) {
        LOG(ERROR) << "Could not read /proc/mounts";
    }
```
- 两个Fstab类对象，fstab读取默认fstab的配置，而mounts对象从/proc/mounts文件读取系统已经mount的文件系统；
- fstab对象读取的默认fstab配置，可以参考ReadDefaultFstab，定义在system/core/fs_mgr/fs_mgr_fstab.cpp文件中，大约是读取了如下文件：
    - 从dt设备树读取；
    - 从/etc/recovery.fstab文件获取；
    - 多个位置，格式为${fstab}<fstab_suffix>,${fstab}<hardware>,以及${fstab}<hardware.platform>；
    - 上述的${fstab}可能包含值："/odm/etc/fstab."，"/vendor/etc/fstab."，"/system/etc/fstab."，"/first_stage_ramdisk/system/etc/fstab."，"/fstab."，"/first_stage_ramdisk/fstab.";
```
    static const std::vector<std::string> kPartitionNames = {"system_ext", "product"};

    android::fs_mgr::Fstab extra_fstab;
    for (const auto& name : kPartitionNames) {
        if (GetEntryForMountPoint(&mounts, "/"s + name)) {
            // The partition is already mounted.
            continue;
        }

        auto system_entries = GetEntriesForMountPoint(&fstab, "/system");
        for (auto& system_entry : system_entries) {
            if (!system_entry) {
                LOG(ERROR) << "Could not find mount entry for /system";
                break;
            }
            if (!system_entry->fs_mgr_flags.logical) {
                LOG(INFO) << "Skipping mount of " << name << ", system is not dynamic.";
                break;
            }

            auto entry = *system_entry;
            auto partition_name = name + fs_mgr_get_slot_suffix();
            auto replace_name = "system"s + fs_mgr_get_slot_suffix();

            entry.mount_point = "/"s + name;
            entry.blk_device =
                android::base::StringReplace(entry.blk_device, replace_name, partition_name, false);
            if (!fs_mgr_update_logical_partition(&entry)) {
                LOG(ERROR) << "Could not update logical partition";
                continue;
            }

            extra_fstab.emplace_back(std::move(entry));
        }
    }
```
- 利用/proc/mounts文件获取的mounts对象，检测/system_ext和/product是否已经mount，如果已经mount直接跳出for循环；
- 如果fstab对象需要mount /system路径的场景下，检查mount点的属性是否逻辑分区，只有逻辑分区才会mount；
- 根据出来的/system挂载点构造entry对象（包含路径，设备），并加入到extra_fstab中；
- 可以看到，system_ext和product路劲的挂载情况会影响system路径的挂载行为，关于上面分析的system_ext, system， product，可以参考[共享系统映像](https://source.android.com/docs/core/architecture/partitions/shared-system-image?hl=zh-cn);
```

    SkipMountingPartitions(&extra_fstab, true /* verbose */);
    if (extra_fstab.empty()) {
        return;
    }

    BlockDevInitializer block_dev_init;
    for (auto& entry : extra_fstab) {
        if (access(entry.blk_device.c_str(), F_OK) != 0) {
            auto block_dev = android::base::Basename(entry.blk_device);
            if (!block_dev_init.InitDmDevice(block_dev)) {
                LOG(ERROR) << "Failed to find device-mapper node: " << block_dev;
                continue;
            }
        }
        if (fs_mgr_do_mount_one(entry)) {
            LOG(ERROR) << "Could not mount " << entry.mount_point;
        }
    }
}
```
- SkipMountingPartitions在system/core/fs_mgr/fs_mgr_fstab.cpp定义，会根据/system/system_ext/etc/init/config/skip_mount.cfg文件的配置，确定是否跳过某些节点的mount，并且将其从extra_fstab中删除；
- 如果extra_fstab中还有需要mount的节点，首先access检查设备是否可访问，然后将其初始化为dm设备，并调用fs_mgr_do_mount_one完成mount操作；
#### 初始化日志
```
SelinuxSetupKernelLogging();
```
而SelinuxSetupKernelLogging实现如下：
```
void SelinuxSetupKernelLogging() {
    selinux_callback cb;
    cb.func_log = SelinuxKlogCallback;
    selinux_set_callback(SELINUX_CB_LOG, cb);
}
```
- 一行代码调用，实现也很简单，只有3行，就是声明了一个selinux的日志回调函数，然后使用selinux_set_callback进行注册；
- 这个selinux_set_callback定义在external/selinux/libselinux/src/callbacks.c文件中，从这个目录名可以看到，专门有一套selinux框架处理selinux相关事项；
- 注册的SelinuxKlogCallback函数，会根据日志的级别进行不同的处理，如果是SELINUX_AVC级别，调用的是SelinuxAvcLog（通过netlink写audit日志），其他级别日志，这是通过KernelLogger记录日志（通过/proc/kmsg文件记录）；
#### 加载sepolicy
```
    LOG(INFO) << "Opening SELinux policy";
    PrepareApexSepolicy();
```
- 我们都知道，selinux要起作用，必须依赖policy，而且不同的policy，赋予了selinux不同的行为，所以selinux本身是机制，而policy是实现管理的策略；
- PrepareApexSepolicy函数在加载policy前做一些前期的准备工作，以及init加载policy的具体事项可以参考注释，写的很清晰；
```
// Updatable sepolicy is shipped within an zip within an APEX. Because
// it needs to be available before Apexes are mounted, apexd copies
// the zip from the APEX and stores it in /metadata/sepolicy. If there is
// no updatable sepolicy in /metadata/sepolicy, then the updatable policy is
// loaded from /system/etc/selinux/apex. Init performs the following
// steps on boot:
//
// 1. Validates the zip by checking its signature against a public key that is
// stored in /system/etc/selinux.
// 2. Extracts files from zip and stores them in /dev/selinux.
// 3. Checks if the apex_sepolicy.sha256 matches the sha256 of precompiled_sepolicy.
// if so, the precompiled sepolicy is used. Otherwise, an on-device compile of the policy
// is used. This is the same flow as on-device compilation of policy for Treble.
// 4. Cleans up files in /dev/selinux which are no longer needed.
// 5. Restorecons the remaining files in /dev/selinux.
// 6. Sets selinux into enforcing mode and continues normal booting.
//
```
- 而PrepareApexSepolicy函数，就是完成上述的1和2步骤，即校验zip文件并将文件解压到/dev/selinux;
```
    // Read the policy before potentially killing snapuserd.
    std::string policy;
    ReadPolicy(&policy);
    CleanupApexSepolicy();

    auto snapuserd_helper = SnapuserdSelinuxHelper::CreateIfNeeded();
    if (snapuserd_helper) {
        // Kill the old snapused to avoid audit messages. After this we cannot
        // read from /system (or other dynamic partitions) until we call
        // FinishTransition().
        snapuserd_helper->StartTransition();
    }

    LoadSelinuxPolicy(policy);

    if (snapuserd_helper) {
        // Before enforcing, finish the pending snapuserd transition.
        snapuserd_helper->FinishTransition();
        snapuserd_helper = nullptr;
    }
```
- ReadPolicy函数从/dev/selinux中将策略文件读入到policy这个字符串变量中；
- CleanupApexSepolicy函数删掉/dev/selinux文件，即将解压的policy删除；
- LoadSelinuxPolicy函数将读取出来的写入到/sys/fs/selinux/load，从而实现真正的policy加载；
- 需要注意的是，/sys/fs/selinux目录需要挂载成selinuxfs，否则是没有load文件存在的；
- 这中间掺杂了snapuserd_helper->StartTransition()和snapuserd_helper->FinishTransition()，相当于一个上下文的保护，具体为何要保护，后续分析snapuserd的时候再分析；
```
// This restorecon is intentionally done before SelinuxSetEnforcement because the permissions
    // needed to transition files from tmpfs to *_contexts_file context should not be granted to
    // any process after selinux is set into enforcing mode.
    if (selinux_android_restorecon("/dev/selinux/", SELINUX_ANDROID_RESTORECON_RECURSE) == -1) {
        PLOG(FATAL) << "restorecon failed of /dev/selinux failed";
    }

    SelinuxSetEnforcement();

    // We're in the kernel domain and want to transition to the init domain.  File systems that
    // store SELabels in their xattrs, such as ext4 do not need an explicit restorecon here,
    // but other file systems do.  In particular, this is needed for ramdisks such as the
    // recovery image for A/B devices.
    if (selinux_android_restorecon("/system/bin/init", 0) == -1) {
        PLOG(FATAL) << "restorecon failed of /system/bin/init failed";
    }

    setenv(kEnvSelinuxStartedAt, std::to_string(start_time.time_since_epoch().count()).c_str(), 1);
```
- 注释很清晰说明了，在进入enforcement模式前，需要restorecon一次上下文；
- 写/sys/fs/selinux/enforce文件，进入enforcement模式；
- 重新切换到/system/bin/init的用户态上下文；
- 设置环境变量，记录selinux生效时间；
#### 跳转init第二阶段
```
    const char* path = "/system/bin/init";
    const char* args[] = {path, "second_stage", nullptr};
    execv(path, const_cast<char**>(args));
```
- exec执行init进程，传递的参数是second_stage，进入init的second stage；
- 至此，selinux加载阶段结束，正式进入init第二阶段；
### init第二阶段
- 参考main.cpp文件中，有init进程的main函数入口中，根据传入的参数进入不同的启动阶段，在SetupSelinux阶段传递的是second_stage参数，所以会调用SecondStageMain函数，进入init第二阶段；
- SecondStageMain函数定义在system/core/init/init.cpp文件中，下面就结合代码分析一下init第二阶段流程；
- 首先与前两个阶段一样，都是安装复位信号处理函数、重定向stdio、日志初始化、设置PATH环境变量等；
#### 设置复位函数
```
trigger_shutdown = [](const std::string& command) { shutdown_state.TriggerShutdown(command); };
```
- 多了一个trigger_shutdown函数指针的赋值，这是一个lambda表达式，意思就是trigger_shutdown接受一个const std::string& command参数，并将command传递调用shutdown_state.TriggerShutdown（command）函数；
#### SIGPIPE处理
```
// Init should not crash because of a dependence on any other process, therefore we ignore
    // SIGPIPE and handle EPIPE at the call site directly.  Note that setting a signal to SIG_IGN
    // is inherited across exec, but custom signal handlers are not.  Since we do not want to
    // ignore SIGPIPE for child processes, we set a no-op function for the signal handler instead.
    {
        struct sigaction action = {.sa_flags = SA_RESTART};
        action.sa_handler = [](int) {};
        sigaction(SIGPIPE, &action, nullptr);
    }
```
- 单独给SIGPIPE信号安装了处理函数，而这个处理函数就是啥都不干，应该可以简化为signal(SIGPIPE, SIG_IGN)一行代码；
#### oom killer设置
```
    // Set init and its forked children's oom_adj.
    if (auto result =
                WriteFile("/proc/1/oom_score_adj", StringPrintf("%d", DEFAULT_OOM_SCORE_ADJUST));
        !result.ok()) {
        LOG(ERROR) << "Unable to write " << DEFAULT_OOM_SCORE_ADJUST
                   << " to /proc/1/oom_score_adj: " << result.error();
    }
```
- 设置init进程的oom_score_adj为-1000，这已经是oom_score_adj的最小值了，那么init进程就永远不会被oom killer误杀了；
#### 会话秘钥
```
// Set up a session keyring that all processes will have access to. It
    // will hold things like FBE encryption keys. No process should override
    // its session keyring.
    keyctl_get_keyring_ID(KEY_SPEC_SESSION_KEYRING, 1);
```
- 关于秘钥，我也不懂，只知道这是一个session key；
#### 创建启动标志
```
    // Indicate that booting is in progress to background fw loaders, etc.
    close(open("/dev/.booting", O_WRONLY | O_CREAT | O_CLOEXEC, 0000));
```
- 作用是创建/dev/.booting文件，类似于一个标记，其他的background fw loaders看到有这个文件，就知道正在启动过程中；
#### adb root
```
    // See if need to load debug props to allow adb root, when the device is unlocked.
    const char* force_debuggable_env = getenv("INIT_FORCE_DEBUGGABLE");
    bool load_debug_prop = false;
    if (force_debuggable_env && AvbHandle::IsDeviceUnlocked()) {
        load_debug_prop = "true"s == force_debuggable_env;
    }
    unsetenv("INIT_FORCE_DEBUGGABLE");

    // Umount the debug ramdisk so property service doesn't read .prop files from there, when it
    // is not meant to.
    if (!load_debug_prop) {
        UmountDebugRamdisk();
    }
```
- 注释也说的很清楚了，主要是在设备unlocked的时候，是否允许adb root；
- 获取环境变量INIT_FORCE_DEBUGGABLE；
- 判断设备是否已经解锁，定义在system/core/fs_mgr/libfs_avb/fs_avb.cpp文件中，具体是否unlocked，分别尝试从dtb、properties以及cmdline中获取；
- 只有设备处于unlocked状态，并且INIT_FORCE_DEBUGGABLE环境变量为true，才会允许设备adb root；
- 如果不允许adb root，将/debug_ramdisk节点umount掉；
#### property初始化
```
void PropertyInit() {
    selinux_callback cb;
    cb.func_audit = PropertyAuditCallback;
    selinux_set_callback(SELINUX_CB_AUDIT, cb);

    mkdir("/dev/__properties__", S_IRWXU | S_IXGRP | S_IXOTH);
    CreateSerializedPropertyInfo();
    if (__system_property_area_init()) {
        LOG(FATAL) << "Failed to initialize property area";
    }
    if (!property_info_area.LoadDefaultPath()) {
        LOG(FATAL) << "Failed to load serialized property info file";
    }

    // If arguments are passed both on the command line and in DT,
    // properties set in DT always have priority over the command-line ones.
    ProcessKernelDt();
    ProcessKernelCmdline();
    ProcessBootconfig();

    // Propagate the kernel variables to internal variables
    // used by init as well as the current required properties.
    ExportKernelBootProps();

    PropertyLoadBootDefaults();
}
```
- 调用PropertyInit()进行初始化，定义在system/core/init/property_service.cpp文件中；
- 首先安装selinux的回调处理函数；
- 创建/dev/__properties__目录；
- CreateSerializedPropertyInfo()函数尝试从selinux的哥哥配置文件中读取selinux的配置，并写入到/dev/__properties__/property_info文件，最终restorecon生效；
- __system_property_area_init()就是创建一片共享内存，映射/dev/__properties__/property_info文件；
- property_info_area.LoadDefaultPath()函数就是映射了/dev/__properties__/property_info文件；
- 三个process前缀的函数，分别从dtb、cmdline以及bootconfig中获取property；
- ExportKernelBootProps()函数将上述process获取的property export出来；
- PropertyLoadBootDefaults()函数又从一些配置文件中获取了一些property；
#### umount
```
    // Umount second stage resources after property service has read the .prop files.
    UmountSecondStageRes();

    // Umount the debug ramdisk after property service has read the .prop files when it means to.
    if (load_debug_prop) {
        UmountDebugRamdisk();
    }
```
- 注释写的很清楚，上一步的property初始化完成后，就可以将相应的挂载点umount了；

#### mount
```
static void MountExtraFilesystems() {
#define CHECKCALL(x) \
    if ((x) != 0) PLOG(FATAL) << #x " failed.";

    // /apex is used to mount APEXes
    CHECKCALL(mount("tmpfs", "/apex", "tmpfs", MS_NOEXEC | MS_NOSUID | MS_NODEV,
                    "mode=0755,uid=0,gid=0"));

    // /linkerconfig is used to keep generated linker configuration
    CHECKCALL(mount("tmpfs", "/linkerconfig", "tmpfs", MS_NOEXEC | MS_NOSUID | MS_NODEV,
                    "mode=0755,uid=0,gid=0"));
#undef CHECKCALL
}
```
- 这部分代码很简单，就是把/apex和/linerconfig挂载成tmpfs；
#### selinux
```
    // Now set up SELinux for second stage.
    SelabelInitialize();
    SelinuxRestoreContext();
```
- selinux的初始化，和启动流程没有太多关系，加上selinux里面东西太多，暂时先不管；
#### 开启propertyService
```
    Epoll epoll;
    if (auto result = epoll.Open(); !result.ok()) {
        PLOG(FATAL) << result.error();
    }

    // We always reap children before responding to the other pending functions. This is to
    // prevent a race where other daemons see that a service has exited and ask init to
    // start it again via ctl.start before init has reaped it.
    epoll.SetFirstCallback(ReapAnyOutstandingChildren);

    InstallSignalFdHandler(&epoll);
    InstallInitNotifier(&epoll);
    StartPropertyService(&property_fd);
```
- StartPropertyService开启了一个新线程PropertyServiceThread，线程里使用了上面创建的epoll，然后和init进程进行socket通信；
#### 启动时间戳和环境变量
```
    // Make the time that init stages started available for bootstat to log.
    RecordStageBoottimes(start_time);

    // Set libavb version for Framework-only OTA match in Treble build.
    if (const char* avb_version = getenv("INIT_AVB_VERSION"); avb_version != nullptr) {
        SetProperty("ro.boot.avb_version", avb_version);
    }
    unsetenv("INIT_AVB_VERSION");
```
- 记录secondStage的启动时间；
- 将INIT_AVB_VERSION环境变量设置到"ro.boot.avb_version"属性后，删除INIT_AVB_VERSION变量；看注释是说OTA使用，具体怎么使用以后再拓展；
#### property设置
```
    fs_mgr_vendor_overlay_mount_all();
    export_oem_lock_status();
    MountHandler mount_handler(&epoll);
    SetUsbController();
    SetKernelVersion();
```
- fs_mgr_vendor_overlay_mount_all()函数首先获取vndk的属性，然后"/system/vendor_overlay/"和 "/product/vendor_overlay/"加上vndk属性作为overlayfs挂载；
- export_oem_lock_status()函数首先获取"ro.oem_unlock_supported"属性，然后根据"ro.boot.verifiedbootstate" property是否为orange设置"ro.boot.flash.locked"，关于oem unlock，可以参考[锁定和解锁引导家在程序](https://source.android.com/docs/core/architecture/bootloader/locking_unlocking?hl=zh-cn)
- SetUsbController()函数通过扫描/sys/class/udc目录下的子目录，设置"sys.usb.controller"这个属性；
- SetKernelVersion()函数通过uname获取内核版本号，设置到"ro.kernel.version"这个property；
#### 内置函数
```
    const BuiltinFunctionMap& function_map = GetBuiltinFunctionMap();
    Action::set_function_map(&function_map);
```
- 设置的这个内置函数，看起来就是一些shell的命令，应该是提供了shell内置命令的功能；
#### mount namespace
- 只能从代码看到有很多mount动作和namespace的建立，虽然注释很多，但还是没有搞懂为什么要做这个事情；
- 要详细了解为什么要做这个事情，需要了解各个目录和对应的进程，才能更加深刻理解这些mount namespace；
#### selinux上下文
```
    void InitializeSubcontext() {
    if (IsMicrodroid()) {
        LOG(INFO) << "Not using subcontext for microdroid";
        return;
    }

    if (SelinuxGetVendorAndroidVersion() >= __ANDROID_API_P__) {
        subcontext.reset(
                new Subcontext(std::vector<std::string>{"/vendor", "/odm"}, kVendorContext));
    }
}
```
- 如果是Microdroid，就不适用selinux的上下文，关于Microdroid，参考[Microdroid](https://source.android.com/docs/core/virtualization/microdroid?hl=zh-cn);
- 剩下的就是__ANDROID_API_P__的版本，建立selinux的上下文；
#### 解析启动脚本
```
    ActionManager& am = ActionManager::GetInstance();
    ServiceList& sm = ServiceList::GetInstance();

    LoadBootScripts(am, sm);
```
- 都知道init进程会解析很多.rc启动脚本，并且执行里面的动作，终于在这个阶段开始执行了；
- 从LoadBootScripts函数传参来看，启动脚本里面应该有action和service两种类型的动作；
- 如果“ro.boot.init_rc” property指定了启动脚本，那么就从制定的路径去加载.rc脚本；
- 否则，系统默认一些.rc脚本路径，详细可以参考LoadBootScripts实现；
- 但是这个函数看起来只是parse了启动脚本，并没有执行.rc脚本里面的动作;
#### GSI设置
```
 // Make the GSI status available before scripts start running.
    auto is_running = android::gsi::IsGsiRunning() ? "1" : "0";
    SetProperty(gsi::kGsiBootedProp, is_running);
    auto is_installed = android::gsi::IsGsiInstalled() ? "1" : "0";
    SetProperty(gsi::kGsiInstalledProp, is_installed);
    if (android::gsi::IsGsiRunning()) {
        std::string dsu_slot;
        if (android::gsi::GetActiveDsu(&dsu_slot)) {
            SetProperty(gsi::kDsuSlotProp, dsu_slot);
        }
    }
```
- 注释已经说了，在.rc脚本运行前需要先保证GSI status可用；
- 关于GSI，可以参考[通用系统映像](https://developer.android.com/topic/generic-system-image?hl=zh-cn);
- 设置gsi的property；
- 从"/metadata/gsi/dsu/active"文件获取dsu状态，并设置property；
- 关于dsu，可以参考[动态系统更新](https://developer.android.com/topic/dsu?hl=zh-cn)；
- 至于为什么gsi要在.rc运行前可用，猜测是.rc脚本里面的action或者service可能会用到这两个property；
#### early-init
```
    am.QueueBuiltinAction(SetupCgroupsAction, "SetupCgroups");
    am.QueueBuiltinAction(SetKptrRestrictAction, "SetKptrRestrict");
    am.QueueBuiltinAction(TestPerfEventSelinuxAction, "TestPerfEventSelinux");
    am.QueueBuiltinAction(ConnectEarlyStageSnapuserdAction, "ConnectEarlyStageSnapuserd");
    am.QueueEventTrigger("early-init");
```
- 首先是cgroup，重点是从/etc/cgroups.json文件mount cgroup挂载点和生成/dev/cgroup_info/cgroup.rc；
- SetKptrRestrictAction函数主要是设置/proc/sys/kernel/kptr_restrict接口，控制用户态是否可以查看内核态符号地址；
- TestPerfEventSelinuxAction函数注释说了，测试内核是否为perf_event_open安装了Selinux的hook，如果存在设置sys.init.perf_lsm_hooks这个property；
- ConnectEarlyStageSnapuserdAction主要是测试snapuserd和snapuserd_proxy这两个早期service是否存在；
-  am.QueueEventTrigger("early-init")将以上几个加入queue的action标记为是early-init阶段；
#### init 
```
    // Queue an action that waits for coldboot done so we know ueventd has set up all of /dev...
    am.QueueBuiltinAction(wait_for_coldboot_done_action, "wait_for_coldboot_done");
    // ... so that we can start queuing up actions that require stuff from /dev.
    am.QueueBuiltinAction(SetMmapRndBitsAction, "SetMmapRndBits");
    Keychords keychords;
    am.QueueBuiltinAction(
            [&epoll, &keychords](const BuiltinArguments& args) -> Result<void> {
                for (const auto& svc : ServiceList::GetInstance()) {
                    keychords.Register(svc->keycodes());
                }
                keychords.Start(&epoll, HandleKeychord);
                return {};
            },
            "KeychordInit");

    // Trigger all the boot actions to get us started.
    am.QueueEventTrigger("init");
```
- 同early-init阶段一样，将wait_for_coldboot_done_action和"KeychordInit"对应的lambda加入queue，并标记为init阶段；
#### late-init
```
    // Don't mount filesystems or start core system services in charger mode.
    std::string bootmode = GetProperty("ro.bootmode", "");
    if (bootmode == "charger") {
        am.QueueEventTrigger("charger");
    } else {
        am.QueueEventTrigger("late-init");
    }
```
- 根据"ro.bootmode" property获取当前是否在充电，并标记是charger还是late-init阶段；
#### main loop
```
    // Run all property triggers based on current state of the properties.
    am.QueueBuiltinAction(queue_property_triggers_action, "queue_property_triggers");

    // Restore prio before main loop
    setpriority(PRIO_PROCESS, 0, 0);
    while (true) {
        // By default, sleep until something happens. Do not convert far_future into
        // std::chrono::milliseconds because that would trigger an overflow. The unit of boot_clock
        // is 1ns.
        const boot_clock::time_point far_future = boot_clock::time_point::max();
        boot_clock::time_point next_action_time = far_future;

        auto shutdown_command = shutdown_state.CheckShutdown();
        if (shutdown_command) {
            LOG(INFO) << "Got shutdown_command '" << *shutdown_command
                      << "' Calling HandlePowerctlMessage()";
            HandlePowerctlMessage(*shutdown_command);
        }

        if (!(prop_waiter_state.MightBeWaiting() || Service::is_exec_service_running())) {
            am.ExecuteOneCommand();
            // If there's more work to do, wake up again immediately.
            if (am.HasMoreCommands()) {
                next_action_time = boot_clock::now();
            }
        }
        // Since the above code examined pending actions, no new actions must be
        // queued by the code between this line and the Epoll::Wait() call below
        // without calling WakeMainInitThread().
        if (!IsShuttingDown()) {
            auto next_process_action_time = HandleProcessActions();

            // If there's a process that needs restarting, wake up in time for that.
            if (next_process_action_time) {
                next_action_time = std::min(next_action_time, *next_process_action_time);
            }
        }

        std::optional<std::chrono::milliseconds> epoll_timeout;
        if (next_action_time != far_future) {
            epoll_timeout = std::chrono::ceil<std::chrono::milliseconds>(
                    std::max(next_action_time - boot_clock::now(), 0ns));
        }
        auto epoll_result = epoll.Wait(epoll_timeout);
        if (!epoll_result.ok()) {
            LOG(ERROR) << epoll_result.error();
        }
        if (!IsShuttingDown()) {
            HandleControlMessages();
            SetUsbController();
        }
    }
```
- 这是一个死循环，正常init进程就在这个循环里面处理后续的事项了；
- 将进程优先级恢复到默认的0，CFS的默认优先级别；
- 探测当前是否被标记shutdown阶段，如果是的话，去处理复位相关事情；
- 使用ActionManager机制执行队列中的第一个action或者service，关于ActionManager机制留到后面再看；
- 至此，各种service启动，SecondStage阶段就相当于结束了；
