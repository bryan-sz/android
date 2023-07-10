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
- CHECKCALL宏应该是一个老司机的手法，融合了多个c语言的宏技巧；
