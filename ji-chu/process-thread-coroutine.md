---
description: >-
  进程、线程的概念在学习操作系统基本原理时，大家都有一定的掌握，协程前面也已经简单介绍了。这些理论基础是重要的，但是还不够。落地一项技术会涉及到具体平台相关的内容，协程实现也不例外。这一节我们结合Linux操作系统，来详细了解下进程、线程的真面目，以及如何利用Linux、c库提供的基础设施来实现协程。
---

# 进程, 线程, 协程

服务编程模型一节，我们介绍了几种常见类型的任务、常见的服务编程模型，以及不同类型任务适合选用什么样的服务编程模型来解决。我们讲libmill主要关注的还是网络IO密集型的问题，在深入了解协程实现之前，我还是尽量把进程、线程、协程的关系再深入剖析一下，方便读者朋友理解，后面我们实操（比如撸一个简单的协程库），或者阅读libmill的源码时不至于懵。

### 进程：资源分配的最小单位

进程，是资源分配的最小单位，操作系统分配内存、cpu、fd等各种资源给进程，进程可以决定如何将这些资源与其创建的子进程或者线程进行共享。

以Linux系统为例，创建进程的方式主要有两种：

* fork，通过fork系统调用可以创建当前进程的一个副本（子进程），父进程、子进程的内容完全一样，只是父子进程中可能会执行相同代码中的不同分支而已。当然了，为了提高内存访问的效率，子进程创建之初与父进程共享某些资源，如子进程访问内存采用写时复制（copy on write）技术，只在写入内存时才会发生父进程内存数据到子进程内存空间的拷贝动作，如果是只读，那会直接共享访问父进程内存地址空间中的数据。
* exec，通过exec系统调用来加载目标程序的代码、数据来替换当前进程的代码、数据，并执行。

进程退出的方式有这么几种：

* 执行完成，正常退出；
* 收到SIGTERM、SIGKILL外部信号，杀进程；
* 遇到异常，如段错误segmentation fault、除0异常等；

如果子进程被kill，但是父进程没有显示地通过wait系统调用向内核确认子进程消亡，那么内核会为子进程保留其进程表中的表项（pid），这样的进程在进程表中存在，但是实际上其之前分配的其他资源已经被释放，这样的进程也不可能被调度器scheduler重新调度执行，所以就将这样的进程称之为“僵尸进程“。

进程在Linux中通过结构体`task_struct`来描述，它就是操作系统原理中经常提及的进程控制块PCB，task\_struct结构体包含的成员字段非常多，大致上包括了内存、cpu调度、名字空间等相关的一些资源信息，当然还有很多其他信息，如子进程列表、信号处理函数等等，只要跟进程相关的能想得到的东西，几乎都在这个结构体里面有描述。

`task_struct`的更多信息，请见：[https://github.com/torvalds/linux/blob/master/include/linux/sched.h](https://github.com/torvalds/linux/blob/master/include/linux/sched.h)

### 线程：任务调度的最小单位

线程，在Linux中是通过轻量级进程LWP来实现的，进程通过系统调用`clone`来创建线程，并指定clone选项来告知内核新创建的LWP与原来的进程在哪些资源上可以实现共享，如内存地址空间、打开的文件描述符、名字空间等等。

线程，在操作系统原理里面被称作是任务调度的最小单位，从Linux实现来看确实也是这样，下面我们结合Linux CFS调度器来说明下线程的调度。在设计上，Linux实际上是采用可调度实体`sched_entity`来表征这一最小调度单位。

每个进程task\_struct都包含了一个sched\_entity，线程实际上是LWP，当然也有包含该sched\_entity。如果一个进程是多线程程序，那么每个线程都是独立参与任务调度，整个进程相比单线程的进程会获得相对更多的调度机会。最初的cfs实现确实是这样的，现在的cfs实现，同一个用户下创建的进程也是这样的，但是这种实现隐约感觉有种安全风险，有没有感觉到？

* 最初的cfs实现，假如多个用户登录，其中一个用户只有1个进程，另一个用户却有100个进程，如何保证不同用户各自50%的cpu占用呢？
* 同一个用户创建的进程，一个进程1个线程，另一个进程100个线程，如何保证两个进程各50%的CPU占用呢？

这两点都是会明显导致CFS不那么公平的因素，那现在Linux中CFS实现是如何解决的呢？

* **sched\_entity抽象可调度任务实体，进程、线程、task\_group**
* **task group实现group scheduling，可以自由组织group，按用户级组织，按任务性质组织；**

Linux里面提供了task group，sched\_entity可以描述一个特定进程的任务调度，也可以描述一组进程的任务调度，如nginx进程启动时会创建一个master和四个workers，这四个workers就属于一个task group统一调度，而不是作为四个独立的任务实体被调度\[2\]。其实这种方式属于cfs group scheduling的范畴\[3\]，cfs调度器先按照user进行划分，每个user有自己的一个task group，所有其创建的进程都在这个task group下，sched\_entity也可以用来描述task group，这样cfs选择一个sched\_entity来运行时发现选择出来的entity不是一个真正的进程，而是一个task group，这个时候怎么办呢？每个task\_group又通过rbtree单独维护了一个runqueue，然后再从这个runqueue中按照cfs调度选择算法选择出一个最高优先级的entity，该entity可能还不是一个有效进程（比如用户显示创建了一些task group，如\[2\]中提及的示例multimedia），那就重复上述过程，直到找到一个真正的进程来执行。

文末列出了部分参考文献供读者查看，如果想了解更详细的任务调度细节，还是要阅读内核源码，推荐《深入Linux内核架构》。

### 协程：用户级轻量级任务实体

一个线程的栈空间大小是有限制的，以Linux为例，线程栈空间大小为2MB，如果内存空间大小一定的话，能够创建的线程数量是有限的。其实，系统能够支持的线程数量，除了内存以外，和处理器也有关系，如果读者了解过GDT/LDT，那应该能体会到笔者这里的意思。

这里我们不讨论如何计算一个系统到底能同时支撑多少个线程，我们是想引出，线程栈2MB对内存空间的要求是比较高的，如果想支持大量级的并发处理，是不可能通过无限制创建线程来解决的。那我们如何解决这类问题呢？

以高性能网络服务开发而言，其多属于IO密集型任务，对网络IO进行非阻塞处理是一个常见的思路。可以配合多进程、多线程来处理，其实单线程也可以应付的来性能方面的问题，比较有代表性的是多进程的nginx、单进程单线程的redis。除了性能，还有些其他的非功能性指标，如能否做到负载均衡、进程监控、可用性等，基于reactor网络模型的变体proxy-worker-controller也是一种不错的思路。

但是，归根究底，写事件回调（如IO多路复用、实时信号驱动、异步IO等）还是不利于可读性、维护性、开发编码，我们是希望编写的代码能够按照事情发展顺序流水账似的展开，从头写到尾，中间不要做一些跳转来跳转去的操作。

还是拿网络IO操作来做例子，协程化的处理可以理解成网络IO代码是“同步编码，异步运行”，代码在执行序上会在网络IO的位置执行任务切换（去执行其他协程），这背后的网络IO则是异步执行的，等网络IO完成后再恢复原来停下的协程继续执行。

协程coroutine，纤程fiber，其实都是一个意思，所表达的都是一种相比线程thread更加轻量级的可调度任务实体，它的栈空间可能是固定的，也可能是可以动态伸缩的。在某些比较简化版的协程库实现中，协程栈大小是固定的，如腾讯ServerBench PlusPlus（简称SPP）协程栈大小固定为128KB，如果在协程中使用了比较大的数据结构或者函数调用链比较深则可能触发访存错误。

libmill作为一个go风格协程库的简单实现，它也是不支持栈空间动态伸缩的，libdill则是在libmill基础上的进一步升级，它支持协程栈大小的动态伸缩，能够适应更多应用场景，在生产环境中使用时，libdill则更值得选择。本书出于学习目的，就以libmill来作为学习材料了，背后的原理也是大同小异的。



参考文献：

\[1\] [Marty Kalin](https://opensource.com/users/mkalindepauledu), fair-scheduling-linux, [https://opensource.com/article/19/2/fair-scheduling-linux](https://opensource.com/article/19/2/fair-scheduling-linux)

\[2\] task group extensions to CFS, [https://www.kernel.org/doc/Documentation/scheduler/sched-design-CFS.txt](https://www.kernel.org/doc/Documentation/scheduler/sched-design-CFS.txt)

\[3\] cfs group scheduling, [https://lwn.net/Articles/240474/](https://lwn.net/Articles/240474/)

\[4\] process containers, [https://lwn.net/Articles/236038/](https://lwn.net/Articles/236038/)

\[5\] linux kernel scheduler basics, [https://josefbacik.github.io/kernel/scheduler/2017/07/14/scheduler-basics.html](https://josefbacik.github.io/kernel/scheduler/2017/07/14/scheduler-basics.html)



