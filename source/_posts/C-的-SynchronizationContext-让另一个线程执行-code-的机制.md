---
title: 'C# 的 SynchronizationContext: 让另一个线程执行 code 的机制'
date: 2017-10-20 22:53:32
tags: 
  - C#
  - SynchronizationContext
  - multi-thread
  - GUI
  - async
---


## 了解 SynchronizationContext 有何价值
SynchronizationContext 是 C# GUI 编程中, 实现 "令后台工作线程更新 UI" 功能的必备知识. 
同时, `async/await` 机制也用到了 SynchronizationContext.
SynchronizationContext 也是理解 async/await 的预备知识.

当然, 标题中的 "A 线程让 B 线程执行一段 code"指的不是 
"A 线程创建 B 线程, 同时给 B 线程指定一个 method to run" 这样基础的东西. 


## SynchronizationContext 
拥有什么能力: "marshal code from thread A to thread B, let thread B execuate the code".
解决什么问题:
最经典的例子: background worker thread 通过 SynchronizationContext, 让 UI 线程 update UI.
(Windows 上, 所有的 UI 资源只能在 UI 所在的线程执行 update, 否则程序就会崩)

后台 thread 更新 UI 是非常常见而且必须的操作. 以前的办法有:
利用 Control.BeginInvoke (in WinForm) Dispatcher.BeginInvoke (WPF).

不过, 设计上 "we never want to have a UI object reference within your bussiness layer".
(有了 UI object reference 之后, bussiness layer 就和特定的 UI Object 类型绑定了, 从而对 UI 的 library 产生了依赖)

```cs
public partial class Form1 : Form
{
    public Form1()
    {
        InitializeComponent();
    }

    private void mToolStripButtonThreads_Click(object sender, EventArgs e)
    {
        int id = Thread.CurrentThread.ManagedThreadId;
        Trace.WriteLine("mToolStripButtonThreads_Click thread: " + id);

        // 把 GUI 线程的 SynchronizationContext 拿来传递给后台工作线程
        SynchronizationContext uiContext = SynchronizationContext.Current;

        // create a thread and associate it to the run method
        Thread thread = new Thread(Run);
        thread.Start(uiContext);
    }

    private void Run(object state)
    {
        int id = Thread.CurrentThread.ManagedThreadId;
        Trace.WriteLine("Run thread: " + id);

        SynchronizationContext uiContext = state as SynchronizationContext;
        for (int i = 0; i < 1000; i++)
        {
            // normally you would do some code here
            // to grab items from the database. or some long
            // computation
            Thread.Sleep(10);

            // 利用 SynchronizationContext.Post, 使得 UpdateUI 函数在 GUI 线程得到执行
            uiContext.Post(UpdateUI, "line " + i.ToString());
        }
    }

    /// This method is executed on the main UI thread. 目的达成
    private void UpdateUI(object state)
    {
        int id = Thread.CurrentThread.ManagedThreadId;
        Trace.WriteLine("UpdateUI thread:" + id);
        string text = state as string;
        mListBox.Items.Add(text);
    }
}
```

"marshal code to another thread" 没有 "魔法":
什么是 marshal: Computations often need to move data from one site to another, and don't have any shared memory.
实际上, 这是 **另一个 thread 主动地支持该功能** 才达成的.	
UI 的 Invoke 机制是通过 Windows message 队列(它是 thread-safe 的)来实现信息传递.


## 其他实现 "后台 worker thread 更新 UI" 的机制
+ MFC: `SendMessage(), PostMessage()`
+ WinForm: `Control.BeginInvoke()`
+ WPF: `Dispatcher.BeginInvoke()`


## Reference
Understanding SynchronizationContext 三部曲; 只看的 Part-I, 已经足以理解; 后面讲如何自己实现其派生类.
+ http://www.codeproject.com/Articles/31971/Understanding-SynchronizationContext-Part-I
+ http://www.codeproject.com/Articles/32113/Understanding-SynchronizationContext-Part-II
+ http://www.codeproject.com/Articles/32119/Understanding-SynchronizationContext-Part-III


