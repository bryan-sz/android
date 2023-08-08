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
## 相关数据结构
- 在开始ueventd_main函数之前，我们先熟悉几个ueventd进程相关的数据结构，对于我们理解ueventd_main函数会更有帮助；
### struct Uevent
- 定义在system/core/init/uevent.h头文件中，表示的是uevent事件：
```
struct Uevent {
    std::string action;
    std::string path;
    std::string subsystem;
    std::string firmware;
    std::string partition_name;
    std::string device_name;
    std::string modalias;
    int partition_num;
    int major;
    int minor;
};
```
### class UeventHandler
- 定义在system/core/init/uevent_handler.h头文件中，是对uevent事件处理的抽象；
```
class UeventHandler {
  public:
    virtual ~UeventHandler() = default;

    virtual void HandleUevent(const Uevent& uevent) = 0;

    virtual void ColdbootDone() {}
};
```
- 重点是HandleUevent函数，接收一个Uevent事件，并进行处理，这是一个虚函数，需要具体的Handler进行实现；
### class DeviceHandler
- 定义在system/core/init/devices.h文件中，继承了UeventHandler类；
```
class DeviceHandler : public UeventHandler {
  public:
    friend class DeviceHandlerTester;

    DeviceHandler();
    DeviceHandler(std::vector<Permissions> dev_permissions,
                  std::vector<SysfsPermissions> sysfs_permissions, std::vector<Subsystem> subsystems,
                  std::set<std::string> boot_devices, bool skip_restorecon);
    virtual ~DeviceHandler() = default;

    void HandleUevent(const Uevent& uevent) override;
    void ColdbootDone() override;

    std::vector<std::string> GetBlockDeviceSymlinks(const Uevent& uevent) const;

    // `androidboot.partition_map` allows associating a partition name for a raw block device
    // through a comma separated and semicolon deliminated list. For example,
    // `androidboot.partition_map=vdb,metadata;vdc,userdata` maps `vdb` to `metadata` and `vdc` to
    // `userdata`.
    static std::string GetPartitionNameForDevice(const std::string& device);

  private:
    bool FindPlatformDevice(std::string path, std::string* platform_device_path) const;
    std::tuple<mode_t, uid_t, gid_t> GetDevicePermissions(
        const std::string& path, const std::vector<std::string>& links) const;
    void MakeDevice(const std::string& path, bool block, int major, int minor,
                    const std::vector<std::string>& links) const;
    void HandleDevice(const std::string& action, const std::string& devpath, bool block, int major,
                      int minor, const std::vector<std::string>& links) const;
    void FixupSysPermissions(const std::string& upath, const std::string& subsystem) const;
    void HandleAshmemUevent(const Uevent& uevent);

    std::vector<Permissions> dev_permissions_;
    std::vector<SysfsPermissions> sysfs_permissions_;
    std::vector<Subsystem> subsystems_;
    std::set<std::string> boot_devices_;
    bool skip_restorecon_;
    std::string sysfs_mount_point_;
};
```
- 类成员比较多，用到的时候再做解读；
### class FirmwareHandler
- 定义在system/core/init/firmware_handler.h头文件中，也继承了UeventHandler，是对Firmware事件的具体实现；
```
class FirmwareHandler : public UeventHandler {
  public:
    FirmwareHandler(std::vector<std::string> firmware_directories,
                    std::vector<ExternalFirmwareHandler> external_firmware_handlers);
    virtual ~FirmwareHandler() = default;

    void HandleUevent(const Uevent& uevent) override;

  private:
    friend void FirmwareTestWithExternalHandler(const std::string& test_name,
                                                bool expect_new_firmware);

    Result<std::string> RunExternalHandler(const std::string& handler, uid_t uid, gid_t gid,
                                           const Uevent& uevent) const;
    std::string GetFirmwarePath(const Uevent& uevent) const;
    void ProcessFirmwareEvent(const std::string& root, const std::string& firmware) const;
    bool ForEachFirmwareDirectory(std::function<bool(const std::string&)> handler) const;

    std::vector<std::string> firmware_directories_;
    std::vector<ExternalFirmwareHandler> external_firmware_handlers_;
};
```
### class ModaliasHandler
- 定义在system/core/init/modalias_handler.h头文件中，也继承了UeventHandler类，是对modalias事件的处理抽象；
```
class ModaliasHandler : public UeventHandler {
  public:
    ModaliasHandler(const std::vector<std::string>&);
    virtual ~ModaliasHandler() = default;

    void HandleUevent(const Uevent& uevent) override;

  private:
    Modprobe modprobe_;
};
```
- 至此，可以看到UeventHandler的处理，包含了DeviceHandler、FirmwareHandler、ModaliasHandler共三类；
### struct UeventdConfiguration
- 定义在system/core/init/ueventd_parser.h头文件中，是对ueventd中配置相关的抽象；
```
struct UeventdConfiguration {
    std::vector<Subsystem> subsystems;
    std::vector<SysfsPermissions> sysfs_permissions;
    std::vector<Permissions> dev_permissions;
    std::vector<std::string> firmware_directories;
    std::vector<ExternalFirmwareHandler> external_firmware_handlers;
    std::vector<std::string> parallel_restorecon_dirs;
    bool enable_modalias_handling = false;
    size_t uevent_socket_rcvbuf_size = 0;
    bool enable_parallel_restorecon = false;
};
```
- 主要是在对ueventd.rc配置文件的解析中会用到，表示相关的各项配置，大体与上面的UeventHandler的子类对应，可以分为device、firmware以及modalias相关配置，以及其他相关配置；
### class UeventListener
- 定义在system/core/init/uevent_listener.h文件中，是对uevent时间进行监听的抽象；
```
class UeventListener {
  public:
    UeventListener(size_t uevent_socket_rcvbuf_size);

    void RegenerateUevents(const ListenerCallback& callback) const;
    ListenerAction RegenerateUeventsForPath(const std::string& path,
                                            const ListenerCallback& callback) const;
    void Poll(const ListenerCallback& callback,
              const std::optional<std::chrono::milliseconds> relative_timeout = {}) const;

  private:
    ReadUeventResult ReadUevent(Uevent* uevent) const;
    ListenerAction RegenerateUeventsForDir(DIR* d, const ListenerCallback& callback) const;

    android::base::unique_fd device_fd_;
};
```
- 从UeventListener的类方法来看，除了poll监听事件外，另外一大功能就是重新生成uevent事件；
### class ColdBoot
- 定义在system/core/init/ueventd.cpp文件中，主要是ueventd进程在系统coldboot状态时相关处理的抽象；
```
class ColdBoot {
  public:
    ColdBoot(UeventListener& uevent_listener,
             std::vector<std::unique_ptr<UeventHandler>>& uevent_handlers,
             bool enable_parallel_restorecon,
             std::vector<std::string> parallel_restorecon_queue)
        : uevent_listener_(uevent_listener),
          uevent_handlers_(uevent_handlers),
          num_handler_subprocesses_(std::thread::hardware_concurrency() ?: 4),
          enable_parallel_restorecon_(enable_parallel_restorecon),
          parallel_restorecon_queue_(parallel_restorecon_queue) {}

    void Run();

  private:
    void UeventHandlerMain(unsigned int process_num, unsigned int total_processes);
    void RegenerateUevents();
    void ForkSubProcesses();
    void WaitForSubProcesses();
    void RestoreConHandler(unsigned int process_num, unsigned int total_processes);
    void GenerateRestoreCon(const std::string& directory);

    UeventListener& uevent_listener_;
    std::vector<std::unique_ptr<UeventHandler>>& uevent_handlers_;

    unsigned int num_handler_subprocesses_;
    bool enable_parallel_restorecon_;

    std::vector<Uevent> uevent_queue_;

    std::set<pid_t> subprocess_pids_;

    std::vector<std::string> restorecon_queue_;

    std::vector<std::string> parallel_restorecon_queue_;
};
```
- 至此，ueventd重点的数据结构都了解的差不多了，就可以开始进入ueventd的最重要的功能了； 

## ueventd_main
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
- 下面的这一部分代码，就要解析uevent.rc配置文件，并正式接收uevent事件进行处理；
