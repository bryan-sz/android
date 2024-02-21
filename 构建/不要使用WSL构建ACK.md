# 不要使用WSL环境构建ACK
我尝试在WSL运行的ubuntu-18.04中构建ack主线代码，结果在第一部make menuconfig阶段就直接失败了。
失败原因是WSL的本身的缺陷，对于Linux中的elf二进制文件格式的支持有问题，导致ack库中prebuilt的clang工具链无法运行。
具体错误和原因可以参考：https://groups.google.com/g/android-llvm/c/xrX4q3FGcZY
