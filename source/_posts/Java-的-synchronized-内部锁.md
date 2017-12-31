---
title: 'Java 的 synchronized, 内部锁'
date: 2017-12-31 23:38:13
tags: 
  - multithreading
  - Intrinsic Lock
  - Monitor Lock
  - Monitor
---


## 基本
`synchronized` == 内部锁 (Intrinsic Lock) == 监视器锁 (Monitor Lock / Monitor)


## 性质, 能力
有三个方面重要性质:
+ 互斥 (mutex) 能力
+ 可见性 (visibility) 能力
+ 可重入性 (re-entrant)

### 互斥能力 + 可见性
(内部锁关于互斥和可见性的性质见 "内存模型" 笔记)

### 可重入, 可重入性 (re-entrant, re-entrancy) 
用于描述同步工具的性质. Java 的内置锁 `synchronized` 和 C# 的 `lock() {}` 语句都是可重入的.
含义是: 某个线程如果试图获得一个已被该线程获得的锁, 这个请求会成功.
```java
class Example extends FatherClass {
    // synchronized 是可重入的 => 调用 foo() 不会死锁;
    // 多线程下的递归的代码, 可能会用到 '可重入' 的锁;
    public synchronized void foo() {
        bar();
    }
    public synchronized void bar() {
    }
    
    public synchronized void doSomething() {
       System.out.println("I am doing something ...");
       // 如果 synchronized 不是可重入的, 这里就会死锁
       super.doSomething();    
    }
}
```
相比之下, C/C++ 的许多互斥锁 (如 pthread 的 mutex) 通常是没有可重入性;

"获取粒度" 的概念: (per-thread acquire 和 per-invocation acquire)
我们常常说, 有 "可重入性" 的锁的获取 **粒度** 是 "线程" (per-thread acquire), 
而没有 "可重入性" 的锁的获取粒度是 "调用" (per-invocation acquire).

"可重入性" 有时会带来意想不到的奇怪行为.

### 与 `volatile` 的比较
`synchronized` 比 `volatile` 功能更强; 不仅可以实现可见性, 
由于有 mutex 能力, 可以让复合操作拥有 "原子性";

```java
public synchronized boolean AddIfNotExist(Element e) {
    if (container.find(e) == false) {
        container.add(e);
        return true;
    }
    return false;
}
```
上面的例子是经典的实现 "check then execute" 逻辑的代码;
在多线程情况下, 即使 container 成员本身自身有线程安全性, 
如果 "check then execute" 没有用恰当的同步机制保证原子性, 代码的整体逻辑依旧会错.

这里用 `synchronized` 实现了 "原子性"; 但如果其他操作 container 的代码没有用同样
正确地用 `synchronized` 保护 (参见下文), 那么 "原子性" 可能被破坏 (或这说 '不成立').


## 正确使用 `synchronized`; 什么是 "用 (内部) 锁保护某状态变量";
首先要理解 "用 (内部) 锁保护某状态变量" 术语, 其含义:
==等于== 对于所有访问某状态变量的代码, **都** 用 **同一个** 锁保护起来
==等于== 访问某状态变量之前, **都** 必须先持有 **同一个** 锁.

**都** 和 **同一个** 这两点都是重点, 否则起不到保护作用:
+ 对于某个状态变量, 只要有一个关于它的操作未被锁保护, 那么那个操作就是不安全的;
  而且大多数时候, 这会导致其他操作的安全性也被破坏
+ 被不同的锁保护的代码, 相互之间绝无互斥和可见性的保证; 
  这是 Java 内存模型决定的
   
第二, 通常涉及同一个 "不变性条件 (invariant)" 的多个变量, 会用同一个锁来保护.
例如, 一个容器的 `size` 和 `isEmpty` 成员逻辑上是强耦合的, 通常会用同一个锁来保护.

第三, 持有锁的时间应该尽可能短; 需要长时间才能完成的操作 (无论是CPU密集还是IO密集),
不应该在持有锁的时候做.

