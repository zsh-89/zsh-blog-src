---
title: '多线程: 线程封闭, 栈封闭, ThreadLocal, 单线程子系统'
date: 2017-12-13 19:52:32
tags: 
  - multithreading 
  - Confinement
  - Thread Confinement
  - Stack Confinement
  - Thread Local
  - single‐threaded subsystem
---

## 线程封闭 (Thread Confinement)
线程封闭 (Thread Confinement) 是一个让减少 "shared mutable state" 的技术/方法.
让尽可能多的 mutable state 成为 "某个线程专有的 mutable state".

线程封闭常见类型:
+ Ad-hoc 线程封闭: 维护 "线程封闭性" 的职责完全交给程序员, 而不提供系统的, 机制上的, 框架上的辅助
+ 栈封闭 (Stack Confinement): 多用局部变量, 让尽可能多的对象的引用不超出局部作用域
+ Java/C# 的 ThreadLocal 类, 语言的库/机制提供的 per Thread 的对象存储; 
+ 单线程子系统 (single-threaded subsystem)


## 单线程子系统 (single-threaded subsystem)
顾名思义, 可知概念. 
最经典的应用在 GUI. 所有的 GUI 框架都有一个单线程子系统:
由一个的专门的 '事件分发线程' (Event Dispatch Thread, EDT) 来处理 GUI 事件 
(也称为 '事件队列' 模型).

"多线程的 GUI 工具包" 被称为 "计算机科学史上的又一个失败的梦想".
许多人尝试过编写多线程的 GUI 框架, 最终, 都因为竟态条件和死锁而回归了事件队列模型.

MVC 设计模式的广泛应用, 更使得 "多线程的 GUI 工具包" 变得不可能实现, 
因为 MVC 导致了更强的访问顺序的不确定性.

GUI 中的表现层对象 (Presentation Object) 只允许在事件处理线程 (也常被称为 UI 线程) 中访问, 
完全无需担心同步问题; 在事件处理线程之外, 不允许访问.

"在 GUI 的事件处理线程之外, 需要让 GUI 执行某样操作" 这个问题的解决思路是: 
+ 通过消息机制通知 GUI 事件处理线程, 事件处理线程根据消息类别主动调用相关的函数
+ 在前一者的基础上, 传递更多的数据, 包括函数的地址和参数等等, 
  将可以描述计算任务的信息打包传递给事件处理线, 让事件处理线程执行
  C# 的 BeginInvoke, SynchronizationContext, Java Swing 的 Runnable 类型都属于这一类.
  好处是: 在前者的基础上, 将事件处理线程和具体响应事件的代码逻辑更好地解耦.
  
不宜把慢的事情 (计算密集 or I/O密集) 交给事件处理线程. 毕竟只有一个线程在做事件处理.

"在 GUI 的事件处理线程让计算线程执行某样操作/任务" 这个问题比较直观, 
如果需要 GUI 和后台计算线程有交互, 可以参考利用 Future/Promise 乃至 Observable 这些异步模型; 
简而言之, 至少需要提供机制保证:
+ UI 库要有能力打断/通知终止计算线程 (用户手动终止任务); 
  C# java 都有提 "取消/终止" 线程的机制/库, 
  如 java 的 `Thread.currentThread().isInterrupted()`, C# 的 `CancellationToken`
+ 计算线程要可以向 UI 报告进度; 
  通常, 计算线程定期算出进度信息, 通过 UI 库提供的机制把进度信息传递给 UI 线程,
  最后由 UI 线程将进度信息展示出来.

