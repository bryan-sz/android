# Android启动（二）- init.rc解析
- 在上一篇文章提到，init进程启动的第二阶段，会解析init.rc文件，然后根据init.rc文件的配置拉起系统的各个daemon和service进程，那么本片文章就聚焦init.rc文件的解析，把这个具体的过程展开；
## init.rc文件位置
- 在源码树中时，位于`system/core/rootdir/init.rc`;
- 在设备上，位于`/system/etc/init/hw/init.rc`;
- 也可能其他位置也可以，详细可以参考上一篇文章[解析启动脚本](https://github.com/bryan-sz/android/blob/main/Android%E5%90%AF%E5%8A%A8.md#%E8%A7%A3%E6%9E%90%E5%90%AF%E5%8A%A8%E8%84%9A%E6%9C%AC)这一节提到的LoadBootScripts函数实现顺序；
```
static void LoadBootScripts(ActionManager& action_manager, ServiceList& service_list) {
    Parser parser = CreateParser(action_manager, service_list);

    std::string bootscript = GetProperty("ro.boot.init_rc", "");
    if (bootscript.empty()) {
        parser.ParseConfig("/system/etc/init/hw/init.rc");
        if (!parser.ParseConfig("/system/etc/init")) {
            late_import_paths.emplace_back("/system/etc/init");
        }
        // late_import is available only in Q and earlier release. As we don't
        // have system_ext in those versions, skip late_import for system_ext.
        parser.ParseConfig("/system_ext/etc/init");
        if (!parser.ParseConfig("/vendor/etc/init")) {
            late_import_paths.emplace_back("/vendor/etc/init");
        }
        if (!parser.ParseConfig("/odm/etc/init")) {
            late_import_paths.emplace_back("/odm/etc/init");
        }
        if (!parser.ParseConfig("/product/etc/init")) {
            late_import_paths.emplace_back("/product/etc/init");
        }
    } else {
        parser.ParseConfig(bootscript);
    }
}
```
- 首先查找"ro.boot.init_rc"这个property，如果没有的话，就按照列出来的顺序一个一个查找，其中第一个位置就是/system/etc/init/hw目录下；
## rc文件语法
- rc文件是Android的启动脚本，Android也专门规定了rc文件的语法，详细说明参考[Android init language](https://android.googlesource.com/platform/system/core/+/master/init/README.md);
## init.rc文件解析
- 因为init.rc配置项太多，我就从我目前关注的启动阶段的事项角度来看最重要的服务是怎么启动的，其余多数都忽略；
### import
- 首先就是import引入其他的rc文件，这样可以解析和启动更多的专项服务；
```
import /init.environ.rc
import /system/etc/init/hw/init.usb.rc
import /init.${ro.hardware}.rc
import /vendor/etc/init/hw/init.${ro.hardware}.rc
import /system/etc/init/hw/init.usb.configfs.rc
import /system/etc/init/hw/init.${ro.zygote}.rc
```
### early-init
```
on early-init
    ...
    start ueventd

    # Run apexd-bootstrap so that APEXes that provide critical libraries
    # become available. Note that this is executed as exec_start to ensure that
    # the libraries are available to the processes started after this statement.
    exec_start apexd-bootstrap
    ...
```
- 可以看到，这个阶段主要是启动ueventd，还有apexd-bootstrap;
### init
```
on init
    ...
    # Start logd before any other services run to ensure we capture all of their logs.
    start logd
    # Start lmkd before any other services run so that it can register them
    write /proc/sys/vm/watermark_boost_factor 0
    chown root system /sys/module/lowmemorykiller/parameters/adj
    chmod 0664 /sys/module/lowmemorykiller/parameters/adj
    chown root system /sys/module/lowmemorykiller/parameters/minfree
    chmod 0664 /sys/module/lowmemorykiller/parameters/minfree
    start lmkd

    # Start essential services.
    start servicemanager
    start hwservicemanager
    start vndservicemanager
    ...
```
- init阶段启动了logd、lmkd、servicemanager、hwservicemanager、vndservicemanager这5个进程；
### charger_mode
```
# Healthd can trigger a full boot from charger mode by signaling this
# property when the power button is held.
on property:sys.boot_from_charger_mode=1
    class_stop charger
    trigger late-init
```
- 通过sys.boot_from_charger_mode这个property判断是否充电模式启动；
### late-init
```
# Mount filesystems and start core system services.
on late-init
    trigger early-fs

    # Mount fstab in init.{$device}.rc by mount_all command. Optional parameter
    # '--early' can be specified to skip entries with 'latemount'.
    # /system and /vendor must be mounted by the end of the fs stage,
    # while /data is optional.
    trigger fs
    trigger post-fs

    # Mount fstab in init.{$device}.rc by mount_all with '--late' parameter
    # to only mount entries with 'latemount'. This is needed if '--early' is
    # specified in the previous mount_all command on the fs stage.
    # With /system mounted and properties form /system + /factory available,
    # some services can be started.
    trigger late-fs

    # Now we can mount /data. File encryption requires keymaster to decrypt
    # /data, which in turn can only be loaded when system properties are present.
    trigger post-fs-data

    # Should be before netd, but after apex, properties and logging is available.
    trigger load_bpf_programs

    # Now we can start zygote.
    trigger zygote-start

    # Remove a file to wake up anything waiting for firmware.
    trigger firmware_mounts_complete

    trigger early-boot
    trigger boot
```
- late-init阶段主要是mount文件系统，这个在注释中有写；
- 还trigger了zygote进程，这个是后续所有java进程的父进程，类似于linux中的init的地位；
- 以下几个阶段都是late-init中的子项了；
#### early-fs
```
on early-fs
    # Once metadata has been mounted, we'll need vold to deal with userdata checkpointing
    start vold
```
- 启动了vold进程；
#### post-fs
```
on post-fs
    exec - system system -- /system/bin/vdc checkpoint markBootAttempt
    ...
    # HALs required before storage encryption can get unlocked (FBE)
    class_start early_hal

    # Load trusted keys from dm-verity protected partitions
    exec -- /system/bin/fsverity_init --load-verified-keys
```
- 启动了vdc、early_hal还有fsverity_init;
---- 后续暂不分析了，搞多了记不住；
