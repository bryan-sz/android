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
- critical表明ueventd是设备关键的服务，同时使用了critical关键字的默认参数，即critical [window=4] [target='bootloader'],
- 
