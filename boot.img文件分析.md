# boot.img介绍
大家都知道android的启动分区划分，包含一个boot分区，对应的镜像就是boot.img，里面包含了启动的时候使用到的kernel, ramdisk以及dtb文件，但是里面具体怎么组织的还是不确定，所以这篇文章就是将boot.img解压缩，查看boot.img的组成；
# boot.img解压
首先，要找到解压boot.img的工具，在构建工程的out/host/linux-x86/bin目录下有构建用到的各种工具，其中mkbootimg是打包boot.img的工具，而对应的解包工具就是unpack_bootimg,使用工具解压缩如下：
```
out/host/linux-x86/bin/unpack_bootimg --boot_img boot.img --out boot
boot magic: ANDROID!
kernel_size: 10548574
kernel load address: 0x40080000
ramdisk size: 11692562
ramdisk load address: 0x47c80000
second bootloader size: 0
second bootloader load address: 0x00000000
kernel tags load address: 0x4bc80000
page size: 2048
os version: 12.0.0
os patch level: 2023-05
boot image header version: 2
product name: 
command line args: bootopt=64S3,32N2,64N2 buildvariant=userdebug
additional command line args: 
recovery dtbo size: 0
recovery dtbo offset: 0x0000000000000000
boot header size: 1660
dtb size: 115534
dtb address: 0x000000004bc80000
chengang15@whws6:~/out/target/product/P98980AA1$ cd boot/


chengang15@whws6:~/out/target/product/P98980AA1/boot$ ls
dtb  kernel  ramdisk
```
- 可以看到，中间会输出boot.img的一些组成信息，然后在指定的输出目录boot下面将boot.img中的dtb、kernel、以及ramdisk都解压了出来；
- 至于boot.img的具体的组织格式，可以留到后面研究再补充；
# dtb
dtb就是设备树文件，但是很奇怪，使用file命令查看的dtb，却提示是data，而不是设备树文件；
```
chengang15@whws6:~/out/target/product/P98980AA1/boot$ file dtb 
dtb: data
```
直接查看dtb的二进制文件，发现在传统的dtb文件头魔术字“d00dfeed”前面还多了64个字节，盲猜是Android在上面又多套了一层；
<img width="304" alt="1692792737453" src="https://github.com/bryan-sz/android/assets/65610624/17a9604b-2a58-4841-8f20-4671bca20cb6">
- 不过不影响，直接dd命令将前64个字节剥离，然后file命令查看，提示是正常的设备树文件了；
```
chengang15@whws6:~/out/target/product/P98980AA1/boot$ file dtb_no_header 
dtb_no_header: Device Tree Blob version 17, size=115470, boot CPU=0, string block size=14782, DT structure block size=100632
```
- 使用工具目录下的dtc命令将dtb反编译成dts文件，就可以查看dts配置了
# kernel
- kernel就没有什么可以查看的了，就是内核的压缩包；
- 实际上好像不是这么一回事儿？ file命令提示是gzip压缩包，然后解压后file命令就是数据了，是不是android又自己加了啥头？
# ramdisk
- ramdisk也是gzip压缩过的，不同于kernel，解压后直接就提示是cpio包了
```
chengang15@whws6:~/out/target/product/P98980AA1/boot$ gzip -d ramdisk_bak.gz 
chengang15@whws6:~/out/target/product/P98980AA1/boot$ file ramdisk_bak 
ramdisk_bak: ASCII cpio archive (SVR4 with no CRC)
```
- cpio再解包一次，就得到我们的树状目录了，可以查看里面完整的内容；
