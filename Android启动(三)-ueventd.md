[TOC]
# Android启动(三)-ueventd
- 在上一篇文章[Android启动（二）- init.rc解析](https://github.com/bryan-sz/android/blob/main/Android%E5%90%AF%E5%8A%A8%EF%BC%88%E4%BA%8C%EF%BC%89-%20init.rc%E8%A7%A3%E6%9E%90.md)中提到，early-init阶段首先启动的就是ueventd；后续其他的进程，也按照这个init.rc文件的解析顺序来分析；
## ueventd是什么
- ueventd进程是AOSP中为了处理kernel的uevent事件而引入的一个daemon进程；当ueventd接收到这样的消息时，它通过采取适当的操作来处理它，这些操作通常可以是在/dev中创建设备节点、设置文件权限、设置selinux标签等。 
Ueventd还可以处理内核请求的固件加载，并为块和字符设备创建符号链接。
## uevend启动
- 在init.rc文件的early-init中有如下定义：
```
on early-init
  ...
  start ueventd
  ...
```
而ueventd的service定义在init.rc文件第1252行：
```
## Daemon processes to be run by init.
##
service ueventd /system/bin/ueventd
    class core
    critical
    seclabel u:r:ueventd:s0
    user root
    shutdown critical
```
- 从上面的service配置可以看出，真正的二进制文件是/system/bin/ueventd;
- ueventd属于core这个class；
- critical表明ueventd是设备关键的服务，同时使用了critical关键字的默认参数，即critical [window=4] [target='bootloader'],意思是如果ueventd在4分钟内或者启动完成前重启了超过4次，设备就会直接重启到bootloader阶段，当然，可以通过设置`init.svc_debug.no_fatal.<service-name>`为true来避免重启；
- seclabel是selinux的策略，这个就不展开了；
- user root就是ueventd属于root用户；
- shutdown critical指定了shutdown时候ueventd在shutdown时不会直接被kill；
## ueventd可执行文件
- 从上面可以看到，二进制是/system/bin/ueventd，我找了一台设备看了一下，发现ueventd是指向init的软链接，这和[Android启动](https://github.com/bryan-sz/android/blob/main/Android%E5%90%AF%E5%8A%A8.md#%E5%90%AF%E5%8A%A8%E8%BF%87%E7%A8%8B)中分析也是一致的；
```
Q706F:/ $ ls -al /system/bin/ueventd
lrwxr-xr-x 1 root shell 4 2008-12-31 19:00 /system/bin/ueventd -> init
```
## ueventd代码分析
- 在system/core/init目录下，有main.cpp文件中，main函数入口：
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
```
service启动的时候没有传递参数，所以argc是1，argv是NULL，直接跳转到uevent的main函数，即ueventd_main函数；
### ueventd_main
- ueventd_main函数定义在system/core/init/ueventd.cpp文件中，就是ueventd的主要文件了；
- ueventd_main函数代码行数不多，只有不到70行，下面就针对进行代码解读；
```
int ueventd_main(int argc, char** argv) {
    /*
     * init sets the umask to 077 for forked processes. We need to
     * create files with exact permissions, without modification by
     * the umask.
     */
    umask(000);

    android::base::InitLogging(argv, &android::base::KernelLogger);

    LOG(INFO) << "ueventd started!";

    SelinuxSetupKernelLogging();
    SelabelInitialize();
```
- 这一部分代码和init进程的FirstStageMain及SecondStageMain类似，都是进行初始化，包括umask，日志以及selinux标签；


