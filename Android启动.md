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
至此，第一阶段init初始化完成，后续分析第二阶段的init；
