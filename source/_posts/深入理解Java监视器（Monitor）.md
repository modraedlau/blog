---
title: 深入理解Java监视器（Monitor）
date: 2019-01-05 10:21:25
tags: Monitor 并发 锁
category: Java并发编程
---

## 管程 ##
我们知道Java中的重量锁是由监视器来实现的，那什么是监视器，监视器又称管程，我们先来看下维基百科上对“管程”的描述：
>管程 (英语：Monitors，也称为监视器) 是一种程序结构，结构内的多个子程序（对象或模块）形成的多个工作线程互斥访问共享资源。这些共享资源一般是硬体装置或一群变数。管程实现了在一个时间点，最多只有一个线程在执行管程的某个子程序。与那些通过修改数据结构实现互斥访问的并发程序设计相比，管程实现很大程度上简化了程序设计。管程提供了一种机制，线程可以临时放弃互斥访问，等待某些条件得到满足后，重新获得执行权恢复它的互斥访问。

## Mark Word ##
有关MarkWord这部分我们也可以打开hotspot源码可以看到markOop.hpp上相关注释，我把注释进行了翻译并摘抄出重点：
```c++
//
//    [JavaThread* | epoch | age | 1 | 01]       lock is biased toward given thread
//    [0           | epoch | age | 1 | 01]       lock is anonymously biased
//
//  - the two lock bits are used to describe three states: locked/unlocked and monitor.
//
//    [ptr             | 00]  locked             ptr points to real header on stack
//    [header      | 0 | 01]  unlocked           regular object header
//    [ptr             | 10]  monitor            inflated lock (header is wapped out)
//    [ptr             | 11]  marked             used by markSweep to mark an object
//                                               not valid at any other time
```
这里我们值得注意的是在锁膨胀为重量锁时，Mark Word锁标志会变成10，而其他部分将会出现指向对应monitor的指针。

## ObjectMonitor ##
另一方面我们从markOop.hpp可以知道其monitor指的就是一个ObjectMonitor，跟踪这点后我们再打开objectMonitor.hpp看下其数据结构：
```c++
  // initialize the monitor, exception the semaphore, all other fields
  // are simple integers or pointers
  ObjectMonitor() {
    _header       = NULL;
    _count        = 0;
    _waiters      = 0,
    _recursions   = 0;
    _object       = NULL;
    _owner        = NULL;
    _WaitSet      = NULL;
    _WaitSetLock  = 0 ;
    _Responsible  = NULL ;
    _succ         = NULL ;
    _cxq          = NULL ;
    FreeNext      = NULL ;
    _EntryList    = NULL ;
    _SpinFreq     = 0 ;
    _SpinClock    = 0 ;
    OwnerIsThread = 0 ;
    _previous_owner_tid = 0;
  }
```
通过查阅一些资料我们知道这里面有几个重要的信息：
* _owner：指向持有ObjectMonitor对象的线程。
* _cxq：FILO竞争队列，应对多线程竞争锁的时候，使用CAS操作替换队列头部。
* _EntryList： cxq中的合适线程可以被放入EntryList，WaitSet中的线程被notify/notifyAll之后，也可能会放入EntryList中，准备竞争锁。
* _WaitSet：线程被wait()后，将会被放入WaitSet中。放入WaitSet中的线程被notify/notifyAll后，将会进入cxq或EntryList中。


画了一张图来理解下：
{% asset_img monitor.png 监视器 %}
---

然后我们可以看下objectMonitor.cpp的核心内容，还好这份源码中也提供了一些注释，我们可以从注释出发，了解其中思路。我把注释进行了翻译和整理：
* 线程成功获取监视器所有权后，使用CAS操作将_owner字段从null更改为非null。
* 原则：任何时候一个线程最多出现在一个监视器列表——cxq、EntryList或WaitSet上。
* 竞争线程使用CAS将自己“推”到cxq上，然后自旋/暂停。
* 竞争线程获得锁后必须从EntryList或cxq中出队。
* 退出监视器的线程会在EntryList上选择一个“假定继承人“标识并取消暂停。注意：退出的线程不会将后继线程与EntryList断开连接。在取消暂停后，被唤醒者将会重新获取监视器所有权。继任者（被唤醒者）将会获得锁定或者重新暂停。
* 继承由竞争性切换策略来决定。退出线程不会将所有权直接授予或者传递给后继线程（这个也称为“切换式”继承）。相反，退出线程释放所有权并可能唤醒后继者，后继者可以（重新）争夺锁的所有权。如果EntryList为空但是cxq不为空，退出线程会将cxq中的元素排入EntryList。他通过分离cxq（通过CAS安装null）并将线程从cxq折叠到EntryList中来实现。EntryList为双向链表，而cxq为单向链表，因为最近到达的线程（RATs）需要基于CAS的操作“推”入队列。
* 并发原则：
  * 只有监视器所有者可以访问或改变EntryList，监视器本身的互斥锁属性可以保护EntryList免受并发干扰。
  * 只有监视器所有者可以分离cxq。
* 监视器的entry list操作可以避免锁，但严格来说他们不是无锁的。进入是无锁的，退出不是。
* cxq可以有多个并发“推送者”但只有一个并发分离线程。这种机制不受ABA问题的影响。更确切地说，基于CAS的“推”到cxq上是不受ABA问题影响的。
* 总之，cxq和EntryList构成线程的单个逻辑队列，试图获取锁。使用两个不同的列表在acquisition之后提高在一定时间内的出队操作几率，并减少列表末端的热度。一个关键的需求是最小化在保持监视器锁时的发生的队列和监视器元数据操作，即使少量的自旋也会大大减少EntryList/cxq上的入队出队操作的数量。也就是说，自旋减轻了对“内部”锁的争用并监视元数据。