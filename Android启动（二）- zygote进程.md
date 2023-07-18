# Android启动（二）- zygote进程
- 在上一篇文章[Android启动](https://github.com/bryan-sz/android/blob/main/Android%E5%90%AF%E5%8A%A8.md)中，已经分析到init进程启动完成，进入了main loop等待，本篇文章详细分析Android中大名鼎鼎的zygote进程；
## zygote进程是什么
- zygote单词本身是受精卵的意思，这个进程名字我觉得还是很传神的，能够比较贴切的表达出这个进程的作用。就是不断的孕育系统中的后续的java进程，所有的java进程都是由zygote进程fork出来的，是后续所有java进程的父进程；
## zygote进程启动
- zygote进程肯定也是需要由其他进程拉起来的，答案就是init进程。
- init进程在[解析启动脚本](https://github.com/bryan-sz/android/blob/main/Android%E5%90%AF%E5%8A%A8.md#%E8%A7%A3%E6%9E%90%E5%90%AF%E5%8A%A8%E8%84%9A%E6%9C%AC)时，就会去查看
- 写到合理，突然想到应该把init.rc文件的解析专门写一下，毕竟init进程是按照init.rc的配置拉起各个daemon和service的，所以zygote先暂停，先写init.rc的解析吧。
