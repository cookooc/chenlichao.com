---
layout: post
title: language-concurrency-implementation
description: language-concurrency-implementation
---

马天夫 (长期无业者) 说：

对javascript不懂；但是thread model vs event model是计算工业由来已久的争论话题。你可以在网上搜索why thread is evil或者why event is evil，都能找到业界顶尖专家的针锋相对的争论。

从编程角度说，如果不考虑性能，如果开发环境提供thread，那就一定要用thread；用thread写出的代码逻辑是连续的和内聚的，没有事件覆盖问题；当然它有死锁问题，而且数学上无法避免。

如果要使用Event Model；系统不能太大，Event路由机制、处理的优先顺序应该简单明了，否则程序的控制流将非常难以理解。在事件路由上挂钩子是应该极力避免的。对并发事件要充分展开，避免出现循环触发。在性能可以接受的情况下尽可能把逻辑正交化，把无关逻辑用periodical polling的机制来处理（包括延迟处理）。每个子系统应该遵循Reactive的设计原则，即当一个外部事件进入时（包括时钟事件），一定要在这个系统内部队列完成全部处理之后再抛出新的事件，采用缓冲延迟的方式。给每个子系统写一个内部事件队列是不错的做法，如果觉得麻烦，用Observer Pattern也是一个办法；抛向全局的事件队列是最后没办法的办法。

事实上任何Event Model的逻辑都可以用从上层（粗粒度系统）到下层的顺序调用来完成，在处理的每个步骤，前一个子系统返回自己的state change提供给后面的子系统使用，后面的子系统可以使用这个state change和访问已经被处理过的前面的子系统的状态更新自己的状态，处理过的子系统的状态保证是consistent的，顺序处理完所有的子系统之后，整个系统的state也是consistent的。你很容易用assertion来验证这一点。

如果一个子系统依赖于多个子系统的状态，这样的做法就会看到所有的并发和多重依赖关系，它体现在你会发现一些子系统的事件处理函数需要把外部事件、多个其他子系统的state change，和另外一些子系统的引用都送进去，看起来有点儿怪？错！这是最美妙的时刻，因为这就是并发情况的显式定义，而且你必须在设计阶段去考虑这种情况，而且你必须在一个子程序内把它搞定（逻辑内聚）。

这对处理并发问题是非常好的，它相当于在设计阶段人肉展开了程序的state space。如果你在设计阶段展开了这个逻辑，定义了所有情况下的处理顺序，这个程序的state就是被穷举的，它的测试工作将非常容易进行；而且系统没有未知的、似是而非的、应该会工作的状态存在。

当然Event Model也不是噩梦，它有特别美妙的一个地方就是你在程序逻辑发生翻天覆地的变化时，你会发现程序的框架竟然基本不需要改动，代码很容易线性增长；但是如果你是为了这个目的而使用Event Model，我只能祝你好运了，兄弟！

---

Rio (编程语言技客) 说：
#
先声明一个基本假设：人的思维是线性的。程序员也是人，思维当然也是线性的。线性这里指的是事件发生顺序的前因后果关系。如果你不认同这个假设，下面讨论的几种风格的区别意义不是很大。

基于回调的异步 I/O 风格（如 Node.js 和 Python Twisted）会导致控制流倒置（inverse of control flow），使得事件发生的先后顺序不清晰明了，从而造成代码的理解和调试困难。在 Node.js 出现之前，广泛使用的 Python Twisted 库大量使用了这样的风格。一般认为这样的代码可维护性很低。基于回调的异步 I/O 的优势在于其开销小、效率高。单线程的架构也避免的多线程修改可变状态的锁的问题（当然单线程也是个限制，能够有效利用多核的方式局限于多进程）。

基于线程的同步 I/O 风格则没有这个问题：每个线程相对独立，且线程内部的控制流是线性的。理解和维护基于线程式的代码相对容易（先不谈锁的问题，不是这里讨论的重点，这里只考虑用于 I/O 的场景）。基于线程的同步 I/O 的问题是它的可扩展性很低，因为每个线程的内存开销大，在线程间切换的开销也大。对于需要处理成千上万连接的网页服务器而言，这样的开销无法接受。

很多人于是尝试保留回调和线程风格的好处（低开销、线性控制流），但同时避免他们的缺点（高开销、倒置控制流）。协程（Coroutine）[1] 可以比较好的解决这个问题。协程允许将某个子程（Subroutine）的运行状态保留下来以便将来重入（re-entry）。也就是说，协程可以随时暂停运行，将控制返回给调用者，等条件成熟时再从暂停点继续运行。基于协程的异步 I/O 和基于回调的异步 I/O 在性能上相当，但因为协程的内部逻辑顺序是线性的，不会导致控制流倒置。主流语言中，Python 的 Generator 和 Ruby 的 Fiber 都是协程的例子。在异步 I/O 的语境下，可以将协程理解为回调模式的语法糖（当然协程还有其他的好处）。基于协程的异步 I/O 风格主要难点在于要自己写调度器（scheduler）在多个协程间切换，而且每个协程要记得频繁让出控制避免其他协程僵死。这和回调风格一样。协程现在还不那么流行，被广泛理解和接受可能还需要一些时间。

单一进程下，单纯基于回调或者协程都无法有效利用多核处理器。条件允许的情况下，一般采用运行多进程进行负载分流。基于线程的方式理论上可以利用多核，但在 Python 和 Ruby 这样有全局解释器锁（Global Interpreter Lock）的官方实现里，通常只能同时运行一个线程，多线程的优势也就局限在线性控制流，负载分流还是得通过运行多进程实现。Python 和 Ruby 的 JVM 实现由于没有全局解释器锁，不存在这个问题，多个线程可以同时运行。

线程当然不是一无是处。问题描述中引用的文献认为线程不好，是在特定的情景下（系统瓶颈主要是 I/O），且当时（2005年以前）多核处理器并非主流、内存相对有限。自从 Linux 2.6 内核开始搭载 NPTL [2] 后，线程的开销（内存、切换）其实已经降低很多了。此外，现在的服务器多核已然是普遍现象，10GB 以上内存也很常见，上万个线程的开销已经不是问题。线程模式可以不用对程序进行特别修改就能利用越来越多的处理器核心（另外一种形式的性能『免费午餐』）。为了优化使用事件驱动的模式而必须进行的状态切换等协调操作进化到最后会成为另外一种形式的线程调度器，某些场合下还不如直接用系统的更加成熟的线程调度器。

像 Erlang [3] 这样采用定制化调度器+轻量级线程的模式也很有意思：Erlang 的线程并非系统线程，而是 Erlang 自己管理的、类似协程的机制。和 Python、Ruby 的协程不同，Erlang 的轻量级线程切换不需要手工管理让出控制。Erlang 的调度器会在某个线程执行一定步骤（Erlang 的语境下称为『缩减』，reduction）后自动切换。这一点更加类似系统线程：内部逻辑是连续的，可以使用同步 I/O，同时又没有系统线程高开销的弊病，可谓一石三鸟。唯一问题是，这个太小众了……

另外值得一题的还有通过多个进程/事件循环提高事件驱动 I/O 性能。比如 Nginx 的工人进程（worker process）。每个工人是单独的事件循环，和其他工人独立。每个工人使用轮询机制（如 poll/epoll）时只需要处理自己手上的套接字，效率相对要高些。多核处理器上，通常为每个核心分配一个工人，这样互相之间不会为抢系统资源打架。
