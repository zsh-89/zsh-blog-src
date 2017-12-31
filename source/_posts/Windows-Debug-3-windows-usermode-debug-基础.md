---
title: 'Windows Debug - 3: windows usermode debug 基础'
date: 2017-12-31 23:45:37
tags:
  - Windows Debugging
  - Windows Usermode Debugging
  - windbg
---

## 前言
不同 debug 场景, 不同 debug 难度.
+ 最简单的是 dev 环境, 可以用 visual studio, 随意增减代码, 打断点, 可以 debug 编译;
+ 最难的 production 环境, 可能只有一个 dump file.

本文说的 debug 技术主要是 "拿不到源码" 情况下的 debug 技术.
目标是尽可能分析出问题的所在, 由于没有源码, 不试图修复问题.
在微软存在专门帮客户分析 production 环境的程序错误的业务.

Windows 操作系统上, 调试 usermode code 的技术:
+ 调试 native code (C, C++)
+ 调试 managed code (C#)
+ Remote debugging


## 词汇术语
+ attach/detach
+ break point/conditional break point
+ debug(checked) build/retail(free) build
+ local process/remote process
+ dump file (memory dump)
+ exception; first-chance exception; second-chance exception
  Windows 操作系统有 structured exception handling (try-except-finally 语句).
  有 exception handler 的称为 first-chance, 没有的则为 second-chance.
  Win32 API: `RaiseException()` 主动 raise.
+ step over, step into, step out, run to cursor
+ symbols, private symbols, public symbols;
  通常是 .pdb 文件.
  包含内容: global var, local var, function name, function entry address, source line-number
+ symbol server
+ call stack
+ register
+ disassembly code
+ user mode / kernel mode
+ thread local storage

## 知识补充
### Windows Virtual Memory 小知识
+ Virtual Memory Type: MEM_IMAGE, MEM_MAPPED, MEM_PRIVATE
+ Virtual Page State: Free, Reserved, Committed, Shared
+ Virtual Memory Protection: PAGE_READONLY, PAGE_GUARD, PAGE_EXECUTE_READ, 等等

### CPU 寄存器
intel 寄存器介绍: https://www.swansontec.com/sregisters.html  The Art of Picking Intel Registers.pdf

### managed code; .Net
在 CLR (common language runtime) 上跑的 code, 称为 managed code, 如 C#.
C/C++ 常称为 native/unmanged code.

### .Net 程序的 Domain 的概念; System Domain, Shared Domain, App Domain
App Domain 介绍:
+ 轻量级的进程, 在进程中的进程
+ 比进程代价低
+ 被 CLR 管理
+ 起到互相隔离作用

System Domain 和 Shared Domain 都是单例的, 每个 managed process 中最多一个; 
分别起到管理和共享 assembly 的作用.

### CLR 简介
IL: Intermediate Language
Manged object;
Manged object method table; 调试时候最经常查看一个 object 的 method table.
```
[C# 或其他 .net manged code 源代码] =====编译==> 
[IL 和 metadata]                   =====runtime 时被执行, 经 JIT 编译==>
[native code]                      =====执行==>
```
不被执行的不会被 JIT 编译; 被 JIT 编译过会有标记.


## 常用 debug 工具
+ WindDbg    调试
+ debugDiag  抓 dump file, 自动分析 dump
+ procDump   抓 dump
+ perfmon    Windows Performance Monitor
+ Visual Studio
+ Profilers, 包括 Visual Studio profiler


## WinDbg 调试 userspace unmanaged code
基本思路
+ 确定出问题的进程 (一般比较容易确定)
+ 根据出现的问题的情况抓 dump (有些问题可能需要抓多个 dump)
+ 要确定出问题的线程 
+ 查看 callstack, 查看 memory usage, 查看程序的汇编代码, 查看 exception, 
  等等, 找出问题所在

```
.cls      清屏
!help <commands>

## unmanaged code 的 callstack 查看
k   display callstack
kc  display only function name
kb  display first 3 params of each funcion
kv  display addtionally frame-type specific info (FPO frame-pointer-ommision info)
kd  display raw stack data
    k 的输出的第一列信息 ('#' 所在列) 就是 frame number

## frame 可以是 call stack 中的一项, 保存一层函数调用相关的所有信息
.frame <frame_number>    change frame
u                        反汇编, current frame 的
    还可以加地址, 例如: u RtlUserThreadStart+0x21
    默认是不断显示更多的, 更后面的 (offset 更大) 的反汇编的结果
dv                       display name/values of local variable of frame
    必须有 private symbol
.cxr <exception record addr>   reset registers to match context record
.ecxr                          switch to faulting context record in dump
~*k                            call stack of all threads
   
## unmanged modules (.dll/.exe 就是一个 module)
x *!             list all modules
x User32!MB*     list all symbols starting with "MB" in user32.dll
lm               list loaded modules   

## break points / 调试命令 
bp <addr>        add break point
    如: bp PowerBall!wmain+0x3a8
bl               list break point
    disable / clear 断点可以通过 bl 输出的 UI 操作
    bd 0: disable 断点 0
g                continue running
r                查看诸多寄存器保存的值
r @eax=1         将 eax 置为 1
```


### 例子: 修改 "抽取 PowerBall (美帝的彩票)" 的小程序
需求:
假设我们设定的 input 是 1 2 3 4 5 6;
(1 - 5 是我们猜测的普通球编号, 6 是我们猜测的 PowerBall 编号).
通过 WinDbg 修改程序 "抽奖" 函数的返回值, 使我们的 input 中奖.

知识点:
+ 使用 x 命令查看 symbol, 寻找问题相关的函数
+ 通过 bp 命令设置断点; 熟悉断点, 执行等调试的命令
+ 通过 r 命令修改寄存器的值, 从而修改函数返回值

#### 修改过程
首先, 用 WinDbg "Open Executable" 打开应用程序, 程序会自动被 WinDbg 中断; 
在我们执行 g 命令前, 程序一直是被中断的状态.

用 x 命令查看 symbol, 找到相关的函数:
```
x *!
x PowerBall!*
x PowerBall!Draw*
    查看到 2 个和 Draw 相关的函数
```

这两个命令可以把断点设在函数入口 (我并不用)
`bp PowerBall!DrawNumber `
`bp PowerBall!DrawPowerBall`

通过 `u PowerBall!DrawNumber` 和继续执行 `u`, 查看汇编代码;
在函数最尾的 ret 语句前用 `bp` 设置断点 (例如: `bp 00aa2ee7`);
同理, 对 `PowerBall!DrawPowerBall` 也在返回语句前配置断点.

g 继续执行程序; 在断点处停下; 通过 `r @eax=1` 把第一次抽奖的结果改为 1 (修改函数的 return value).
重复操作, 把抽奖结果分别改为 "2 3 4 5".
在 `PowerBall!DrawPowerBall` 返回前, 把抽 "PowerBall" 的结果, 用 `r @eax=6` 改为 6.

其他修改思路: 
+ 通过 `x PowerBall!*` 可以看到小程序把我们的输入保存在 `PowerBall!g_LotteryTicket = int [6]` 中. 
也可以把这里面的值修改为 "抽奖" 函数的结果.
+ 找到最后 "判定抽奖成功" 的地方, 修改比较结果



## WinDbg 调试 userspace managed code
调试 C# (或其他 .Net 上的托管的语言) managed code, 首先考虑反编译, 
得到 C# 的源代码, 不需要去学习 IL 指令.

`sos.dll`: general debugging extension for .Net; shipped with .Net;
能够让 windbg 更好地调试 .Net 的代码 (否则只能像调试 native code 一样调试 .net code).
一定要用上和抓 dump 环境同样版本的 sos.dll

WinDbg 的 Open Executable 和 attach 几乎等价; 
一般选择先开 exe 再 attch, 这样有机会做 load sos.dll; 
attach 功能和 DebugDiag 的 "rule 监听" 冲突, 同一时刻只能开一者.

基本思路:
+ 确定出问题的进程 (一般比较容易确定)
+ 根据出现的问题的情况抓 dump (有些问题可能需要抓多个 dump)
+ 要确定出问题的线程 
+ 查看 callstack, 查看 memory usage, 查看程序的汇编代码, 查看 exception, 
  等等, 找出问题所在

查找关注的 method 的方法: 
+ 通过 domain 查到 assembly, 通过 assembly 查到 module (一般是 dll/exe 文件),
+ 通过 module 查到到自己关心的类 (method table); 
+ 通过 method table 查到自己关心的 method; 
+ 定位到 method 后可以去查看反编译的代码, 定位具体问题

```
## 加载 sos.dll
.loadby sos clr
.loadby sos mscorwks
.load <path>\sos

## 通过 domain 查到 assembly, 通过 assembly 查到 module (一般是 dll/exe 文件),
## 通过 module 查到到自己关心的类 (method table); 
## 通过 method table 查到自己关心的 method; 
## 定位到 method 后可以去查看反编译的代码, 定位具体问题
!dumpdomain                    list all domain
!dumpdomain <domain_addr>
!dumpassembly <assembly_addr>
!dumpmodule   <module_addr>

## dump method table 和 method description 最经常用的
!dumpmodule -mt <module_addr>      查看 module 中的所有 method table   
!dumpmt -md <method_table_addr>    查看 method table 中的所有 method 的 description
    method 的状态:
    + PreJIT 预编译
    + JIT    已被编译; 一般执行到才编译
    + NONE   没有被编译
    
!DumpMT /d <method_table_addr>  可以看到 method table 的 EEClass 信息
!dumpclass <eeclass_addr>       查看 EEClass 信息; EEClass 信息并不太多用


!dumpstack    查看 managed 和 unmanaged 的 stack
    -EE       只看 managed 的 stack
!clrstack     查看 managed stack; 比 !dumpstack 不加 EE 要简明得多;
    -p        show argument
    -l        show local variables
    -a        all above
~*e!clrstack  查看所有 managed thread
!eestack      查看所有 thread call stack, 以 dumpstack 的方式
k             查看 unmanaged stack      

!dumpstackobject / !dso
!do <object_addr>     在 !dso 输出结果里面, 用鼠标点就可以
!DumpArray            这些命令通常在 UI 上都可以用鼠标点出来
!dumpheap
    -stat             group by method
!eeheap -gc           显示 heap gc 信息
!address -summary     显示虚拟地址使用情况的统计信息
    managed 程序的内存往往有 <unknown>; 可以作为识别标志
    
!threads              list threads
!ThreadState <thread_id>   也可以直接 UI 上鼠标点

~.                    current thread info
~<number>             thread info, 指定线程编号
~~[<os_thread_id>]    thread info, 指定线程 id
~*                    all threads in the process
~#                    thread that caused the exception

# for live debug; thread command
~e  execute a thread-specific command
~f  freeze thread
~u  unfreeze
~n  suspend
~m  resume 

~~[<os_thread_id>]s   切换当前线程
    os_thread_id == !thread 的输出中的 OSID 列的信息
~<managed_thread_id>s 切换当前线程
    os_thread_id == !thread 的输出中的 ID 列的信息
    
!pe / !PrintException    看 managed exception
    可以用 do! 看 managed exception object 的信息

# 为 manged code 设置 break point; 断点打到 managed method
!bpmd DebugLab.exe DebugLab.Program.CrashLab   
```

### 场景: 查看 exception 导致的 crash
操作:
+ 抓 dump
+ WinDbg 打开 dump
+ load sos.dll
+ !pe 查看 exception object
+ !clrstack 查看 call stack

### 场景: hang 问题
分类 low cpu hang (idle hang), high cpu hang (busy hang);
注意 hang 型问题, dump 至少抓 3 个, 间隔时间由实际问题决定.
例如, 读取资源本来就要 1min, 那么抓 dump 间隔就要超过 1min. 
可以用 procdump 或者自己用 powershell 做到 "间隔时间抓 dump".

DebugDiag Analysis 看可以帮助分析 dump 文件; 例如, 可以统计多少线程在等待.

`!syncblk, !locks` (sos.dll 的命令) 查看正在被等的同步对象/lock, 看到哪个线程持有对象/lock.

### 场景: memory leak
有可能导致 OutOfMemory (OOM) exception, crash;
可以看 Windows Performance Monitor 看内存使用曲线, .net 内存曲线.

mange code 如果也泄漏, 先查 pinvoke 等调用 naitive 代码的地方.

查看是否有 leak 常检验的指标:
+ working set: physical pages owned by a process.
+ Private Bytes: is the current size, in bytes, of memory that 
  this process has allocated that **CANNOT be shared** with other processes.

利用 debugDiag, 先在 'process' tab 选择进程, "monior for leak", 这样抓的 dump 会包含更多细节.
这样的的 dump file 可以在 DebugDiag 的 analysis 中分析出更好的结果, 
在 WinDbg 也可以看到更多信息.

抓之后, WinDbg 里可以用: `!dumpheap`

### 场景: high cpu hang
runaway thread: stuck 的, 类似陷入死循环的 thread

根据条件抓 dump:
`procdump -ma -u -c 24 -s 5 -n 3 -o BaddApp C:\temp`
    -u Treat CPU usage relative to a single core (used with -c)
    -c 24: 4核机器, 超过 24% (有一个核几乎完全被占用) 时抓 dump
    -n 3 抓 3 个
    -o overwrite existing dump
    [Dump_Folder] 是 `C:\temp`
    
在 WinDbg 中, `!runaway` 查看线程用时统计. 比较多个 dump 文件中 `!runway` 的输出,
可以找到有问题的 runway 线程. 
接着切换到对应线程, 查看 callstack 即可.
    
gc 百分比一般不超过 10%. 有时候 gc 也会导致 high cpu hang。
    
## remote debug
user mode 用得较少了(更经常用 remote desktop, 变成 local debug问题), kernal mode 用得多.

```
## 用 WinDbg 在被调试机 (target) 开启服务 (WinDbg 成为 server 机)
.server tcp:port=9999
    命令会输出 <debugger> -remote tcp:Port=9999,Server=NPC
    tcp:Port=9999,Server=NPC 就是之后会用到的 connection string
.server npipe:pipe=<Pipe Name>
    命令会输出: <debugger> -remote npipe:Pipe=xyz,Server=NPC
    npipe:Pipe=xyz,Server=NPC 就是之后会用到的 connection string
如果是跨公网连接, 前的 connection string 中改为 Server=<ip>, 机器必须是公网可见的
    
## 在 host 用 WinDbg 连接到 target:
WinDbg UI: [File]->[Connect To Remote Session], 输入前面得到的 connection string
```

