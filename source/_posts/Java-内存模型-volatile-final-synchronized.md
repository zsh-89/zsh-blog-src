---
title: 'Java 内存模型; volatile, final, synchronized'
date: 2017-12-13 19:28:15
tags: 
  - Memory Model
  - Java Memory Model
  - volatile
  - final
  - synchronized
  - happens-before
  - as-if-serial
  - data race
  - sequential consistency
  - acquire semantics
  - release semantics
---



## 前言, 参考文献
这篇东西的目的是: 做一次比较完整的严谨的调查和总结. 
我自己这方面知识是不少的, 但不系统.

于是, 在开始写这篇之前就不希望参考除了标准文献之外的其他资料. 
毕竟每个作者在写材料的时候都抱着不同的目标, 有些不严谨的简化确实有其用处; 
但是犹如传话游戏, 经过多次层叠引用不是完全准确的信息, 每个人都添加一点自己的主观因素, 
最后重要的规则和概念会变得得面目全非.

唯一拥有定义权的, 一定准确的标准文献只有 Java Language Specification 和 JVM Specification, 
其中与 Memory Model 相关的部分的内容的前身就是 JSR133.  

可惜标准文档并不自我完备, 并没有真的把事情写明白;
看似简要精准的定义, 其实隐含了大量信息, 关键词组/词汇的含义不明, 
例如, "be consistent with" "predictable" 就包含了大量含义, 绝不是只看字面上的英文词汇就能明白的. 

我出发的原意是通过标准文档, 总结:
+ 足以指导我写出正确程序, 分析一切异常行为的规则
+ 重要的工具/概念 (如 volatile) 的性质, 行为

标准文档的完备性令人不满, 所以实际的主要参考资料改为:
+ JSR133 FAQ; 来自起草 JSR133 的 cs.umd.edu
+ Java Concurrency in Practice; 

通过参考这些 "可读" 的资料反过来理解标准中种种定义的含义.

所有参考资料:
+ JSR133: https://jcp.org/en/jsr/detail?id=133 
  标准文献: Java Memory Model and Thread Specification Revision
+ Java Language Specification (Java 语言标准文献) 其中的 Memory Model 章节
+ JSR 133 FAQ: http://www.cs.umd.edu/~pugh/java/memoryModel/jsr-133-faq.html 
+ https://docs.oracle.com/javase/tutorial/essential/concurrency/locksync.html
+ Java Concurrency in Practice
+ 深入理解 Java 虚拟机, 周志明; 


## 硬件平台的 Memory Model
本质上, 任何程序代码都要变为某个硬件平台上的机器指令, 才能被执行.
这些机器指令是用硬件平台上的汇编语言描述的.

现代的多核/多CPU架构的硬件上存在 "缓存一致性 (cache coherence)" 问题:
多个处理器 (核) 有各自的多级 cache, 跑在它们之上的硬件指令都涉及同一块内存时, 
有可能出现各自的 cache 中数据不一致的情况.
这些问题会使得我们虽然有汇编代码, 却无法从中分析出程序的具体行为.
例如, 至少存在以下无法无法绕过的问题:
+ 某个核对内存的写指令, 何时对另一个核可见?
+ 某个核对内存的读指令, 究竟是否读到了当前最新值?

所以说, 各个 CPU/核 对同一块内存的读的效果和写的效果, 都必须严格定义,
否则程序员无法通过汇编代码知道 "同时在多CPU/核上在运行的指令" 的含义/效果/行为.

通过定义 "处理器 (核) 在访问缓存时都需要遵循的读写协议" 或 "内存模型" 来定义执行内存操作的效果.
例如, intel 就为自己的产品给出了相关定义: https://en.wikipedia.org/wiki/Intel_Memory_Model


## java 内存模型 (java memory model(JMM), MM, JMM)
硬件平台的汇编语言需要 memory model, 高级语言同样需要.
否则, 一旦涉及 "同时并发运行的多线程" 代码, 程序员就无法通过分析高级语言代码本身知道其行为.
如果语言标准不规定好内存模型, 那么它的多线程程序就完全没有可移植性; 
其多线程代码的行为, 必须靠分析具体的编译器的实现和程序所在的硬件平台的内存模型才能得知.

java 的内存模型是 java 程序跨平台性的保证.
java 是最早提供内存模型的语言; 之后, C++ 等其他语言也开始注意到内存模型的重要性, 开始完善.

内存模型必须足够严格, 使得读写操作的效果不产生歧义;
内存模型必须足够宽松, 使得虚拟机可以自由地利用软件硬件的加速技术 
(如 CPU 的执行乱序执行, 编译器的指令重排序, 等等).

JDK 1.5 之后, java 的内存模型趋于完善.


## 重要概念
### happens-before 规则 (Happens-Before Order; HB 规则; HB)
标准中的定义: Two actions can be ordered by a happens-before relationship. 
If one action happens-before another, then the first is visible to and ordered before the second. 
记法: If we have two actions x and y, we write hb(x, y) to indicate that x happens-before y.

为什么需要 HB 规则?
```java
// 线程1
{
    this.important_data = 15;
    this.flag = true;
}
// 线程2
{
    while (this.flag == false) {
        Thread.yield();
    }
    System.out.println(this.important_data);
}
```
只看代码, 我们可能期望:
1. 在时序上, "this.important_data 的赋值" (记为 A) 先于 "this.flag 的赋值" (记为 B) 先于 
   "线程2输出 this.important_data" (记为 C)
2. 线程2在输出 this.important_data 时, 可以看到其最新的值, 即 15

不过, 由于编译器优化和CPU硬件指令优化, A B 的时序是不确定的;
编译器的 "指令重拍" 可能令 B 在时序上先于 A 发生. 
所以, 第一点期望未必成立.

假设事件发生的时序确实是 A -> B -> C, 线程2输出的也未必是 15, 因为 A 的结果未必对 C 可见
(C 可能跑在另一个 CPU 核上, 其 cache 中 this.important_data 的值未必有及时得到更新).
所以, 第二点期望未必成立.

显然, 在多线程编程中, 我们有些时候需要足够强的工具, 
让我们能让 "代码上看起来先发生的动作确实先发生", 而且让 "先发生的动作的结果确实地对后面的动作可见"; 
这就是 happens-before 规则的意义.

这里的例子中, 只要保证有 `hb(A, B)` 和 `hb(B, C)`, 程序的行为就能满足我们的期望.

并不是任何时候都用同步工具保证 happens-before 对于每一条语句都成立, 
这是不经济的; 对于非线程间共享的值, 或者 immutable 的值, 完全没有必要.
过于严格的规则相当于禁止了编译器和硬件的优化, 降低性能.

### 线程内 as-if-serial 语义 (within-thread as-if-serial semantic)
同义词:
+ within-thread as-if-serial semantic (as-if-serial 语义)
+ intra-thread semantics

Java 语言标准保证: 在任何时候, 在一同线程内部, 一定满足 as-if-serial 语义, 即: 
Java 编译器, JVM, 硬件可以使用一切优化手段, 包括且不限于调整语句执行顺序, 只要满足: 
在线程内, 程序的执行效果等价于顺序执行每一个语句的结果.

> The compiler, runtime, and hardware are supposed to conspire to create the illusion of as-if-serial semantics, which means
> that in a single-threaded program, the program should not be able to observe the effects of reorderings.
>
> --- By JSR-133 FAQ

"线程内的 as-if-serial 语义" 并不是很强的约束; 例如前面部分的例子, 
仅看线程1, "编译器调换 A B 执行顺序" 完全符合 as-if-serial 语义; 
但一旦考虑线程2的逻辑, "调换 A B 执行顺序" 就完全破坏了我们对代码行为的期望.

### 数据竞争 data race (和 race condition 不同, 虽然有联系); sequential consistency 顺序一致性
Java 标准文档中对 data race 的定义: 
When a program contains two **conflicting accesses** 
(if at least one of the accesses is a write) that are not ordered by 
a happens-before relationship, it is said to contain a data race. 

简单地说, 同时满足这三个条件就存在数据竞争:
+ 一个进程中, 2个或2个以上的线程同时访问同一块内存空间, 
+ 至少一个线程执行写操作 (即存在 **冲突操作**),
+ 这些冲突操作 (没有通过编程语言的机制建立) HB 约束关系 
  (不那么准确的大白话: 这些冲突操作没有正确地同步)

freedom from data race 是相当强的约束, 由于冲突操作间都有 HB 关系, 意味着:
对同一块内存, 每一个读操作都一定能读到前一个写操作的结果 (写可以来自包括其他线程的任意线程).

sequential consistency 等价于 freedom from data race (data race free), 是同义词汇.

### 顺序一致性 sequential consistency / freedom from data race; 保证程序 predictable 的性质
(几乎全引用自标准本身, 晦涩; 参考 `volatile` 的例子才能解读)

标准中的定义 intra-thread semantics (线程内语义): 
+ is the semantics for single-threaded programs
+ allow the **complete prediction of the behavior** of a thread
  based on the values seen by read actions within the thread. 

标准中的定义 program order (of inter-thread actions): 
Total order (of **inter-thread** / **intra-thread** actions) 
that **reflects** the order in which these would be performed 
according to the **intra-thread** semantics.

标准中的定义 sequential consistency: all **inter-thread actions** occur in a total order 
(the execution order) that **is consistent with program order**, and
对同一块内存, 每一个读操作都一定能读到前一个 (时间上最近的前一个) 写操作的结果 
(写可以来自包括其他线程的任意线程).

sequential consistency 等价于 freedom from data race (data race free), 是同义词汇.

sequential consistency 是相当强的约束; 编写多线程程序的程序员会喜欢它, 
但是并不能直接拿它作为一个程序语言的 MM, 因为它限制过多, 阻碍了编译器优化和CPU并发技术. 

标准指出: sequential consistency 虽然强, 但它 still allows errors arising
from groups of operations that need to be perceived atomically and are not.


## Java 内存模型的策略; as-if-serial 的进一步解释
Java 语言是允许出现 data race 的; Java 语言只保证 "线程内 as-if-serial 语义".
如果 Java 语言标准要求 "Java 内存模型" 任何时候都满足 sequential consistency, 
那么非常多的优化技术都不能用了, 这是不经济的.
Java 的做法是, 提供足够的同步工具, 让程序员可以为特定重要的代码自己实现 sequential consistency.

一般未加同步的代码, 我们只能假设: 
在线程内部, 程序运行的效果等价于一步步顺序地执行每一个语句的结果 (as-if-serial) 

"一步步顺序地执行 (serial)" 包括性质:
+ 代码中前面的语句在后面的语句之前执行, 完全按照程序代码中的顺序执行 
+ 后面的语句总是能看到前面的语句执行的结果 (线程内部的 visibility)).

"as-if-serial" 表示程序运行起来, 从线程内部看, 就好像满足以上性质一样.

回头看 HB 部分的例子, 
我们总可以认为, 在线程1中, 语句是顺序执行的;
`this.flag = true` 被执行时, 前一个语句已经被执行了, 而且能看到前一个语句执行的结果.
但是, 由于 `this.flag` 没有使用任何同步机制, 所以线程2并不能看到线程1所看到的, 
"能否看到, 能看到什么, 以什么顺序看到" 几乎都是未定义的 
(给编译器和硬件留下了充分的自由度, 进行指令优化).


## Java 内存模型保证成立的 "HB 关系"
>1. Each action **in a thread** happens before every action in that thread that comes later in the program's order.
>2. An unlock on a monitor happens before every subsequent lock on **that same** monitor.
>3. A write to a volatile field happens before every subsequent read of **that same** volatile.
>4. A call to start() on a thread happens before any actions in the started thread.
>5. All actions in a thread happen before any other thread successfully returns from a join() on that thread.
>
> --- By JSR-133 FAQ

在标准中描述了更多其他 Java MM 保证一定成了的 "HB 关系", 但是 FAQ 写的这 5 条是重点.

注意 "comes later", "subsequent" 这些本身就在描述先后关系的词汇.
例如 "An unlock on a monitor happens before every subsequent lock on that same monitor"; 
如果没有 HB, 可能出现以下行为:
+ 源代码角度看, 更前面的 unlock 的执行时间可能在更后面的 lock 执行时间之前
  (重排序, 乱序执行等可以造成这个效果)  
+ 即使某个 unlock 的执行时间先于后面的 lock, 这个 unlock 执行结果未必对后面那个 lock 可见

逐点解读:
+ 关于 1: 事实上 1 就是保证了 as-if-serial 效果. 这句话在标准中对应的一句话是: 
  "If x and y are actions of the same thread and x comes before y in program order,
then hb(x, y)."
  注意: 措辞上 "in a thread"/"of the same thread" 表示描述的是线程内部的效果; 
  还有, program order 未必是唯一的, program order 只要 "**reflects** the order in which these actions 
  would be performed according to the **intra-thread** semantics." 就可以了
+ 关于 2-3: 描述了 `lock/unlock` 和 `volatile` 的一部分性质; 它们完整的性质将在下文描述
+ 关于 4-5: 符合程序员对于线程行为的基本期望 


## Java 内存模型提供的的基本同步工具
### `volatile` 
volatile 的性质:
+ 保证被 `volatile` 修饰的变量的跨线程可见性 (visibility): 
  所有线程都一定能看到 volatile 共享变量上一次 (最近一次) 被写入的值
+ Anything that was visible to thread A when it writes to volatile field f 
  becomes visible to thread B when it reads f.
  注意, 必须是同一个 volatile 变量 (前文中都是 'f'), 不同的 volatile 变量是没有这样的保证的.

关于第二点, 继续参考 HB 部分的例子. 我们可以认为, 在线程1中, 语句执行效果等价于顺序执行的效果;
`this.flag = true` 被执行时, 前一个语句 `this.important_data = 15` 已经被执行了; 
而且 `this.flag = true` 能看到前一个语句执行的结果.

现在, 由于 `this.flag` 使用了 `volatile` 同步机制, 所以线程2也像线程1一样, 
看到 `this.flag = true` 的结果时, 一定能看到之前的语句 `this.important_data = 15` 的结果.
(准确地说, 实际上 `volatile` 保证的是 "Java 保证线程2运行的效果等价于
 满足 '线程2能先看到 `this.important_data = 15` 的结果, 再看到 `this.flag = true` 的结果' 的情况下
 程序的运行结果";
 只要求结果等价, 不要求严格地照搬规则执行; 不把话说死, 从而就没有把优化的空间封死.
 如果在 `volatile` 前面的语句运行的结果 "完全无关紧要(不影响结果, totally irrelevant)", 
 那么实际的实现可以根本不真正去做
 '让线程2能先看到 `this.important_data = 15` 的结果,  再看到 `this.flag = true`' 这一点)

### `final`
`final` 解决的问题: 早期的 JMM 中, final 的语义约束并没有现在这么强,
有时从其他线程观察, `final` 成员的值会发生变化, 与程序员的期望不一致. 
就是说, "发布 (publish, 即 make visible) 对象的引用" 的指令被重排序, 提前被执行,
而构造函数还没有被执行完, 其他线程看到了 final 成员的 default initial value 或其他中间状态.

现在的 JMM 中, 只要对象的构造 "不发生逸出", `final` 可以保证:
+ the values assigned to the final fields in the constructor will be visible to 
  all other threads without synchronization. 
  即, 只要我们能在某个线程看到对象的引用, 我们就一定能 "正确地" 看到对象的 final 域的值, 
  不需要其他同步措施.
+ the visible values for any other object or array referenced by those final fields 
  will be at least as up-to-date as the final fields
  即, final 域的 "正确性" 是递归地得到保证的; (例如, 假如你的类中有一个 final 数组成员, 
  你除了关心数组的引用本身的 "正确性", 你也关心数组中的每一个元素的 "正确性";
  Java MM 可以保证后一点也正确.)

补充说明: 
+ 构造函数 "没有逸出" 的充要条件:  do not write a reference to the object being
constructed in a place where another thread can see it before the object's constructor
is finished. (来自 java 语言标准, jls9)
(更多可参考 "可见性, 发布和逸出, 发布对象引用.md")
+ "正确地/正确性" = "up to date as of the end of the object's constructor", not "the latest value available". 

(我认为 `final` 的效果也可以用类似定义 `volatile` 一样的方式定义, 
 都是令 "inter-thread 间的 visibility 和 as-if-serial" 做到了原本 
 "intra-thread 的 visibility 和 as-if-serial"  的效果, 符合程序员的期望)

### 内部锁, intrinsic/monitor lock, `synchronized`
intrinsic lock === monitor lock, 又被简称为 monitor. 
`synchronized` 用到的机制就是 monitor.

synchronized block: 被 `synchronized` 保护的代码区域.

`synchronized` 的性质 & 作用:
+ Every object has an intrinsic lock associated with it.
  从代码角度看, 任何 Java Object (假设名字是 `x`) 都可以使用 `synchronized(x) {}` 语句.
+ Synchronization ensures that memory writes by a thread before
  or during a synchronized block are made visible in a **predictable** manner to 
  other threads which synchronize on the **same monitor**. 
  不是同一个 monitor 的, 没有保证. 
  特别地, `synchronized (new Object()) {}` 可以被认为是毫无用处.
  如何 "predictable" 在后面具体描述.
+ After we **exit a synchronized block**, we **release the monitor**.
+ Before we can **enter a synchronized block**, we **acquire the monitor**.
+ A thread is said to **own the intrinsic lock** between the time it has 
  acquired the lock and released the lock. As long as a thread owns an intrinsic lock, 
  no other thread can acquire the same lock. 
  The other thread will **block** when it attempts to acquire the lock.
+ 参考 'Java 内存模型保证成立的 "HB 关系"' 部分的第一条和第二条; 由于存在这样的 HB 关系, 
  Java MM 实际上保证了: 
  Any memory operations which were visible to a thread before exiting a synchronized block are visible to any thread after it enters a synchronized block protected by the **same monitor**; 
  为什么: 
  All the memory operations happen before the release (被第一条保证, 这里没有跨线程), 
  and the release happens before the acquire(被第二条保证, 这里跨了线程).

C# 定义了 [Acquire Semantics 和 Release Semantics](https://docs.microsoft.com/en-us/windows-hardware/drivers/kernel/acquire-and-release-semantics), 
+ An operation has **release semantics** if other processors will see every preceding operation's effect before the effect of the operation itself
+ An operation has **acquire semantics** if other processors will always see its effect before any subsequent operation's effect. 

这和前面提到的 Java 的 monitor 的 acquire 和 release 是一样的.
C# 是通过详细地定义 release 和 acquire 操作而不是通过 HB rule 来解释 `volatile`, `lock() {}` 
等同步工具的能力的. Java 主要通过 HB rule. 这两种渠道都是较为常见的, 都要有所了解.

### `volatile` 和 `synchronized` 的经典案例: "Double-Checked Locking is Broken" 
```java
class Foo { 
  private Helper helper = null;
  public Helper getHelper() {
    if (helper == null) 
      synchronized(this) {
        if (helper == null) 
          helper = new Helper();
      }    
    return helper;
    }
  // other functions and members...
}
```

如果 `helper` 没有 `volatile` 修饰, **这段代码是错误的**. 因为读取到 helper 不为空时, 
构造函数 Helper() 未必执行完了, Java MM 不存在这样的保证.
除了添加 `volatile`, 也可以把整个方法变为 `synchronized`. 

详细讨论见 [The "Double-Checked Locking is Broken" Declaration]( http://www.cs.umd.edu/~pugh/java/memoryModel/DoubleCheckedLocking.html );
里面更详细地讨论了种种看似聪明但不 work 的 "fix 方案".

