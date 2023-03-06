---
layout: post
title:  "Rust Atomics and Locks读后感"
date:   2023-03-06 20:06:06
categories: Rust 内存安全 线程安全 Atomic Lock Send Sync
excerpt: Rust for Linux
---

* content
{:toc}

Rust语言宣称具有内存安全和高性能特点，并提出无惧并发的概念，但究竟何为内存安全?它会涉及哪些方面?在并发和并行计算的环境下是否涉及更多因素?

理论上，内存安全在语言层面能完全解决嘛?还要确保高性能，哪些方面依赖操作系统/内核来提供支持?

![overview memory safety complex](nasa.image.for.memory.safety.png "overview memory safety complex")
对系统编程或Atomic有一定了解的朋友，往往会遇到lock、lock_free、memory ordering、memory barrier、fence、happens-before、synchronizes-with等概念，其中memory barrier和memory ordering的实现还会涉及编译器和处理器，不同编译器与处理器的实现可能会有所不同，这会给大家一个非常庞杂而无法全面把握的印象；

[<font color="blue">Rust Atomics and Locks</font>](https://marabos.nl/atomics/)这本书全面而精简的以Rust编程视角，解读Atomics和Locks涉及的核心知识点，并提供大量示例代码来说明关键概念和优化事项；

推荐对并发编程感兴趣的朋友深入阅读，作者作为Rust标准库维护者具有丰富经验，全面理解相关示例或许会有一定难度，但一定会受益非浅；
现结合这次阅读还有以前对相关主题的理解，记录相关核心知识点和资料，争取对Atomic相关内容能知其然，知其所以然：

A.并行开发领域的Linux内核大牛[<font color="blue">Paul E. McKenney</font>](https://paulmck.livejournal.com/)，以前转发过其文章[<font color="blue">Rust语言应该使用什么样的内存模型?</font>](https://mp.weixin.qq.com/s?__biz=MzIxMTM0MjM4Mg==&mid=2247483875&idx=1&sn=74af1e8851eea45f9cf2b7e959dd7ff1)，这一次还特别为Rust Atomics and Locks写了书评[<font color="blue">Foreword by Paul E. McKenney</font>](https://marabos.nl/atomics/foreword.html)，他本人十多年来都还在持续更新他的书籍<Is Parallel Programming Hard, And, If So,What Can You Do About It?>；

---
B.内存安全从单线程场景来看往往涉及[<font color="blue">RAII</font>](https://en.cppreference.com/w/cpp/language/raii)概念，包括内存申请、初始化、使用引用借用、析构、释放内存相关操作；
这个概念最早由C++社区提出，后来Rust社区在其基础上增加所有权和移动的概念，更加细化对象T使用、引用&T、借用&mut T，包括在编译器引入静态borrow checker和lifetime等来保证内存操作的安全；

---
C.内存安全从多核多线程并行执行场景来看则更加复杂，毕竟至少涉及多核之间原子操作和同步的问题，其相关概念及理念实践其实一直随着计算机产业的发展在演进；

C++社区在C++11规范中首次提出Memory Model和Memory Ordering相关规范，以从语义和概念上实现语言层面上的统一，进而实现跨编译器和处理器的透明，容许不同编译器、操作系统内核和处理器各自发展的空间；

Rust语言基于C++社区定义的Memory Model规范来实现并行并发场景的内存安全；

---
D.Rust社区生态为了无惧并发和内存安全，在语言层面及标准库提出新的Interior Mutability和Send+Sync等概念，并对单线程和多线程场景实现逻辑概念上的统一；

通过使用&T引用可安全修改内容，Cell+RefCell适用于单线程场景，AtomicI32等和Mutex、RwLock适用于多线程场景，它们都基于UnsafeCell来实现，同时遵循borrow check的统一检查，这有别于其他C/C++/Go/Java等语言独特性，广受爱好者的好评；

Rust Atomics and Locks[<font color="blue">第1章Basics of Rust Concurrency</font>](https://marabos.nl/atomics/basics.html)对其有全面介绍，笔者以前的文章[<font color="blue">从独一无二的视角解读Rust内存和线程安全</font>](http://mp.weixin.qq.com/s?__biz=MzIxMTM0MjM4Mg==&mid=2247483902&idx=1&sn=6533c2959e315d184ba48914576ac4b2)可作为补充参考；

---
E.全面准确理解Memory Ordering中定义的Relaxed、Acquire、Release、AcqRel、SeqCst概念相当重要；
其[<font color="blue">C++ Memory Ordering规范参考</font>](https://en.cppreference.com/w/cpp/atomic/memory_order)比较晦涩难懂；

相信阅读完Rust Atomics and Locks的[<font color="blue">第2章Atomics</font>](https://marabos.nl/atomics/atomics.html)和[<font color="blue">第3章Memory Ordering</font>](https://marabos.nl/atomics/memory-ordering.html)及加上第[<font color="blue">4章Building Our Own Spin Lock</font>](https://marabos.nl/atomics/building-spinlock.html)、[<font color="blue">5章Building Our Own Channels</font>](https://marabos.nl/atomics/building-channels.html)、[<font color="blue">6章Building Our Own Arc</font>](https://marabos.nl/atomics/building-arc.html)的示例后，对基本认知应该没啥问题；

---
F.对Memory Ordering概念的理解，如果按分布式系统比如git中同步拉取数据，本地读取修改数据，提交存储数据等思路来理解，会将复杂问题简单化，更轻松一些，但细节会有所不同；

觉得比较重要的细节有：
Relaxed除了可保证原子操作外，还会保证单个原子变量的total modification order，不会对其他共享变量的执行顺序产生约束；

Acquire+Release除了对相应原子变量实现原子操作外，还会对其他非原子共享变量的执行顺序产生约束，并设置同步点，在读取检查成功后达到数据同步的目的，并且往往只在两个线程之间直接产生Happens-Before Chain，多个线程之间未必有严格同步保证；
SeqCst则在Acquire+Release基础上实现对所有原子变量的SeqCst操作在所有线程的全局性一致保证；类似分布式系统中执行每一个操作包括读取和存储都要求与数据中心保持同步和一致性；

---
G.Rust Atomics and Locks[<font color="blue">第7章Understanding the Processor</font>](https://marabos.nl/atomics/hardware.html)从处理器的角度来解释为啥会提出Memory Ordering的概念，主要涉及处理器以pipeline方式执行指令、StoreBuffer、Cache L1+L2+L3等，并介绍Cache MESI协议逻辑及其可能影响原子操作的性能；

最为特别的是它提供了针对同一个原子操作使用不同Memory Ordering在X86_64和Arm64处理器中对应不同的汇编指令实现，让大家对其内在实现有一个直观的对比认知；
![instructions.for.various.atomic.operations](instructions.for.various.atomic.operations.png "instructions overview for atomic operations")

笔者在此基础进行整理，提供一份[<font color="blue">在线对比结果</font>](https://godbolt.org/z/fvnos15sz)，供大家学习参考；

---
H.并发并行计算中对内存进行atomic原子操作是基础，要实现更高级应用往往会需要锁lock，而要实现锁则需要由操作系统/内核的调度机制来提供线程停止执行和可唤醒重新执行的逻辑；
Rust Atomics and Locks[<font color="blue">第8章Operating System Primitives</font>](https://marabos.nl/atomics/os-primitives.html)中针对这些内容分别对Linux、MacOS、Windows进行介绍；还特别使用futex相关概念及接口在[<font color="blue">第9章Building Our Own Locks</font>](https://marabos.nl/atomics/building-locks.html)实现了Mutex、ConVar、RwLock；

---
I.在Rust Atomics and Locks[<font color="blue">第10章Ideas and Inspiration</font>](https://marabos.nl/atomics/inspiration.html)提到并行开发可能涉及其他数据结构、算法、锁的基础逻辑比如Semaphore、RCU、Parking lot、SequenceLock、crossbeam等，这些新的想法和灵感，或能为未来的并发性能优化提供参考；

---
J.记录补充一些与C/C++并发开发相关的资料链接；
>[<font color="blue">Atomic vs. Non-Atomic Operations</font>](https://preshing.com/20130618/atomic-vs-non-atomic-operations/)

>[<font color="blue">Locks Aren't Slow; Lock Contention Is</font>](https://preshing.com/20111118/locks-arent-slow-lock-contention-is/)

>An Introduction to Lock-Free Programming</font>](https://preshing.com/20120612/an-introduction-to-lock-free-programming/)

>[<font color="blue">Memory Ordering at Compile Time</font>](https://preshing.com/20120625/memory-ordering-at-compile-time/)

>[<font color="blue">Memory Barriers Are Like Source Control Operations</font>](https://preshing.com/20120710/memory-barriers-are-like-source-control-operations/)

>[<font color="blue">Acquire and Release Semantics</font>](https://preshing.com/20120913/acquire-and-release-semantics/)

>[<font color="blue">The Happens-Before Relation</font>](https://preshing.com/20130702/the-happens-before-relation/)

>[<font color="blue">The Synchronizes-With Relation</font>](https://preshing.com/20130823/the-synchronizes-with-relation/)

>[<font color="blue">Acquire and Release Fences</font>](https://preshing.com/20130922/acquire-and-release-fences/)

>[<font color="blue">Weak vs. Strong Memory Models</font>](https://preshing.com/20120930/weak-vs-strong-memory-models/)

>[<font color="blue">Relaxed-Memory Concurrency from Peter Sewell</font>](https://www.cl.cam.ac.uk/~pes20/weakmemory/)

>[<font color="blue">Parallel Programming: September 2022 Update from Paul E. McKenney</font>](https://arxiv.org/abs/1701.00854)

>[<font color="blue">ABA problem</font>](https://en.wikipedia.org/wiki/ABA_problem)

>[<font color="blue">Linux Kernel Memory Barriers</font>](https://www.kernel.org/doc/Documentation/memory-barriers.txt)

---
更多文章可使用微信扫公众号二维码查看

![qrcode.2121labs.](/imgs/qrcode_for_gh_07bc06f8b91d_430.jpg "qrcode.2121labs")

