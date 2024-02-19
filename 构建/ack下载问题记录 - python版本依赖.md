# python版本依赖
- 最近在wsl的ubuntu-18.04环境准备下载android的ack主线内核，运行`repo init -u https://android.googlesource.com/kernel/manifest -b common-android-mainline`这个命令时，弹出如下错误：
```
warning: Python 3 support is currently experimental. YMMV.
Please use Python 2.6 - 2.7 instead.
Traceback (most recent call last):
  File "/usr/bin/repo", line 886, in <module>
    main(sys.argv[1:])
  File "/usr/bin/repo", line 854, in main
    _Init(args, gitc_init=(cmd == 'gitc-init'))
  File "/usr/bin/repo", line 340, in _Init
    _CheckGitVersion()
  File "/usr/bin/repo", line 394, in _CheckGitVersion
    ver_act = ParseGitVersion(ver_str)
  File "/usr/bin/repo", line 364, in ParseGitVersion
    if not ver_str.startswith('git version '):
TypeError: startswith first arg must be bytes or a tuple of bytes, not str
```
- 从这个提示来看，是repo命令不支持python3，但是我本地使用的是python3版本，刚好我本地python2.7和python3.6都有安装，于是到/usr/bin目录，将python软链接指向了python2.7，重新运行此命令，但是得到了另外一个错误：
```
Get https://gerrit.googlesource.com/git-repo/clone.bundle
Get https://gerrit.googlesource.com/git-repo
  File "/home/chengang15/android-kernel/.repo/repo/main.py", line 95
    )
    ^
SyntaxError: invalid syntax
```
- 看起来比第一个错误往前近了一些，至少运行到了ack库中的python脚本了，但是还是报了语法错误。想着是不是和我前面改了python版本有关系，将python又改回了2.7版本，重新运行命令，又回到了第一个错误。自此，就死循环了。
- 仔细想了一下，可能是我本地的repo太老了，repo依赖python2，但是ack库中的python脚本又依赖python3，所以造成了这样一个死循环。于是尝试升级repo工具版本，输入`apt-get upgrade repo`，坑爹的来了，提示当前repo已经是ubuntu上最新的repo版本了。
- 没办法，只能将ack库中repo复制到了/usr/bin目录下，手动升级repo，同时将python软链接指向python3.6，自此，问题解决。
