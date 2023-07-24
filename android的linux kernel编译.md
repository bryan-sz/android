# android的linux kernel编译
- 众所周知，android的底层采用的是Linux kernel，而且在构建android镜像的时候会统一把kernel也构建出来，那么kernel在android里面是怎么构建出来的呢？
  - 一是通过查看构建脚本，即build/envsetup.sh脚本，最终确定到构建kernel的步骤，不便之处是因为构建脚本多样化，且过程较多，不是特别直观；
  - 另一个方法是查看构建日志，通过构建日志反推构建过程，能够比较直观的看到编译的过程；
## 日志目录
- 在源码的顶层目录下，构建之后会有一个out目录，专门用于放置构建过程中的日志和构建后的发布件；
- 在out目录下，编译后会有一个verbose.log.gz日志，用gunzip命令解压后就可以查看verbose.log，这个就是构建日志；
## 编译过程
- verbose.log日志是纯文本，有几百M，直接一行一行查看找到内核的编译日志，效率太低；
- 我们知道，不管aosp上层的编译框架如何包装，编译kernel的话，还是要使用make方式，如果是交叉编译平台，还需要使用CROSS_COMPILE参数将工具链传递给make；
- 因此，使用CROSS_COMPILE关键字就可以很快的搜索到内核的编译日志，我搜索到的如下:
```
[4679/58167] /bin/bash -c "./prebuilts/build-tools/linux-x86/bin/make -j48 -C kernel-4.19 O=/home/chengang/aosp/out/target/product/xxx/obj/KERNEL_OBJ ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- LLVM=1 CC=clang LLVM_IAS=1 HOSTCC=clang LD=ld.lld HOSTLD=ld.lld HOSTLDFLAGS=-fuse-ld=lld LD_LIBRARY_PATH=/home/chengang/aosp/prebuilts/clang/host/linux-x86/clang-r383902/lib64:\$LD_LIBRARY_PATH ROOTDIR=/home/chengang/aosp PATH=/home/chengang/aosp/prebuilts/clang/host/linux-x86/clang-r383902/bin:/home/chengang/aosp/:/usr/bin:/bin:\$PATH PROJECT_DTB_NAMES='xxx/xxx' WT_COMPILE_FACTORY_VERSION='' xxx_defconfig userdebug.config"
make: Entering directory '/home/chengang/aosp/kernel-4.19'
make[1]: Entering directory '/home/chengang/aosp/out/target/product/xxx/obj/KERNEL_OBJ'
  GEN     ./Makefile
  HOSTCC  scripts/basic/fixdep
  HOSTCC  scripts/kconfig/conf.o
  YACC    scripts/kconfig/zconf.tab.c
  ...
#
# configuration written to .config
#
make[1]: Leaving directory '/home/chengang/aosp/out/target/product/xxx/obj/KERNEL_OBJ'
make: Leaving directory '/home/chengang/aosp/kernel-4.19'
```
- 从上述日志可以看到，android构建总共58167个子任务，而这次make menuconfig是其中的一个；
```
[58075/58167] /bin/bash -c "(./prebuilts/build-tools/linux-x86/bin/make -j48 -C kernel-4.19 O=/home/chengang/aosp/out/target/product/xxx/obj/KERNEL_OBJ ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- LLVM=1 CC=clang LLVM_IAS=1 HOSTCC=clang LD=ld.lld HOSTLD=ld.lld HOSTLDFLAGS=-fuse-ld=lld LD_LIBRARY_PATH=/home/chengang/aosp/prebuilts/clang/host/linux-x86/clang-r383902/lib64:\$LD_LIBRARY_PATH ROOTDIR=/home/chengang/aosp PATH=/home/chengang/aosp/prebuilts/clang/host/linux-x86/clang-r383902/bin:/home/chengang/aosp/:/usr/bin:/bin:\$PATH PROJECT_DTB_NAMES='mediatek/xxx' WT_COMPILE_FACTORY_VERSION='' ) && (if [ -e out/target/product/xxx/obj/KERNEL_OBJ/arch/arm64/boot/compressed/.piggy.xzkern.cmd ]; then cp out/target/product/xxx/obj/KERNEL_OBJ/arch/arm64/boot/compressed/.piggy.xzkern.cmd out/target/product/xxx/obj/KERNEL_OBJ/arch/arm64/boot/compressed/.piggy.xzkern.cmd.bak; sed -e 's/\\\\\\\\\\\\\\\\/\\\\\\\\/g' < out/target/product/xxx/obj/KERNEL_OBJ/arch/arm64/boot/compressed/.piggy.xzkern.cmd.bak > out/target/product/xxx/obj/KERNEL_OBJ/arch/arm64/boot/compressed/.piggy.xzkern.cmd; rm -f out/target/product/xxx/obj/KERNEL_OBJ/arch/arm64/boot/compressed/.piggy.xzkern.cmd.bak; fi ) && (python device/mediatek/build/build/tools/check_kernel_size.py out/target/product/xxx/obj/KERNEL_OBJ kernel-4.19 mediatek/xxx )"
make: Entering directory '/home/chengang/aosp/kernel-4.19'
make[1]: Entering directory '/home/chengang/aosp/out/target/product/xxx/obj/KERNEL_OBJ'
  GEN     ./Makefile
scripts/kconfig/conf  --syncconfig Kconfig
/home/chengang/aosp/kernel-4.19
  WRAP    arch/arm64/include/generated/asm/bugs.h
  WRAP    arch/arm64/include/generated/asm/delay.h
  WRAP    arch/arm64/include/generated/asm/div64.h
  WRAP    arch/arm64/include/generated/asm/dma.h
  WRAP    arch/arm64/include/generated/asm/dma-contiguous.h
  WRAP    arch/arm64/include/generated/asm/early_ioremap.h
  WRAP    arch/arm64/include/generated/asm/emergency-restart.h
  WRAP    arch/arm64/include/generated/asm/hw_irq.h
  WRAP    arch/arm64/include/generated/asm/irq_regs.h
  WRAP    arch/arm64/include/generated/asm/kdebug.h
  WRAP    arch/arm64/include/generated/asm/kmap_types.h
```
- 而如上，是真正开始编译zImage的地方；
- 再后续，就是编译其他KO的地方；
