---
title: 'Windows Debug - 1: dump file; 收集方式; 配置'
date: 2017-12-31 23:45:35
tags:
  - Windows Debugging
  - dump file
  - windbg
---


## 分类
按照程序所在 mode 分类:
+ usermode dump file
+ kernel mode dump file


## 为什么要收集 dump file?
+ 对于 kernel mode 程序, 一出错基本上就蓝屏, 所以必须通过 dump file 来分析错误原因.
+ 对于 user mode 程序, 有时候不允许工程师远程登录生产环境, 或者不允许工程师拿到源码,
也只能借助 dump file 来分析错误.


## 抓取 usermode dump file
不推荐用 "任务管理器". 
推荐用 debugDiag (可以方便用 GUI 设定生成 dump 的 rule) 或 procDump.
### procDump
```
procdump -ma -u -c 24 -s 5 -n 3 -o BaddApp.exe C:\temp
    -ma           full dump
    -u            Treat CPU usage relative to a single core (used with -c)
    -c            24: 4核机器, 超过 24% (有一个核几乎完全被占用) 时抓 dump
    -n            3 抓 3 个
    -o            overwrite existing dump
    [Dump_Folder] 上面的命令中, dump 文件输出的目录是 `C:\temp`

procdump -e -ma debuglab        ## -e  monitor crash
```
### DebugDiag 
DebugDiag 安装完有好多个 exe. 配置抓取 dump 的 rule, 在 DebugDiagCollection 里做.
DebugDiag 通过一个后台运行的 service 来抓去 dump. 相比之下, procdump 更加轻量级.

DebugDiag 中配置 "dump 抓取的 rule" 时的注意点:
+ action limits: 代表采集几次 dump, 收集几次 log, 等等
+ crash 类型的 dump 基本上就是 second chance exception;
  first change exception 是被程序内部的 exception handler 处理的 exception.

debugDiag 的用法例子参见 debug 实例.


## 抓取 kernelmode dump file
原则: 如果怀疑驱动有问题，早早地直接 dump，免得错误扩散。

### kernel dump file 的级别介绍; 配置 dump file 级别
minimal dump: 几百 K; 
kernel dump: paged out 的 memory 进不了; 
full dump: 内存多大就多大;

配置 dump 的级别:
`[修改环境变量的 dialog]->[Advanced]->[starup and recovery]`

`Kernel memory dump` 一般足够了。
如果 C 盘不够大：搜索 `delegate dump file`

minidump 的位置：`C:\Windows\minidump\*`
内存数据：`C:\Windows\MEMORY.DMP`

### 利用 OS "蓝屏后 dump file 的生成" 的原理; 从 pagefile 中拿 dump file;
原理:
1. 系统把内存信息放到 pagefile 里。
2. 正常启动一次，系统检查 dump 配置，根据配置，把 pagefile 中东西放入 dump file。

结论:
+ 挂掉、关机、正常重启之后才能有 dump 。
+ 如果不能正常再启动了，那就 **手动把 pagefile 拿出来，它就是 dumpfile** 。改下后缀名即可。

[相关问题] 即使 NTFS 文件系统挂了，一样可以生成 dump file?
> 系统有 ntfs_dummy 这个进程，专门负责写 dump pagefile。

### (强制) 蓝屏法; 收集 Kernel mode dump file
强制蓝屏的方式
+ SystemInternal: NotMyFault 强制蓝屏的工具
+ 改注册表，用特殊键盘按键来触发蓝屏。
+ Call kernel 函数: KeBugCheck(), KeBugCheckEx().

### windbg 法; live debug 中, 收集 Kernel mode dump file
用 Windbg `dump` 命令直接生成dump (系统不停住)。
这样生成的 **dump 文件不好**, 因为系统并没有真正停住, 收集到的不是纯粹的 snapshot。

所以, 要发 dump 给 support 时，要说明白是通过这个办法生成的。
**推荐用蓝屏法来 dump**。
