---
description: >-
  libmill是Martin Sustrik发起的一个go风格的c语言协程库，讲到这里，可能大家对Martin
  Sustrik比较陌生，但是对ZeroMQ应该还算了解吧。鼎鼎大名的ZeroMQ之父，正是Martin
  Sustrik。这一c语言协程库实现代码量1w+左右，但是其设计、实现之精巧，充分展示了作者的深厚功力。学习这样的协程库设计实现真的是非常享受的一件美事。
---

# libmill协程库

### libmill项目

libmill是一个面向c语言的协程库，其下载地址、文档可以在这里找到：[libmill](http://libmill.org/)， 其源代码托管在github上，点击这里查看：[libmill-source](https://github.com/sustrik/libmill)。您也可以通过 [sourcegraph](https://sourcegraph.com/github.com/sustrik/libmill?utm_source=chrome-extension) 在线阅读。

### go风格api

libmill协程库参考了go的设计，其编程接口与go非常接近，如下图所示。

![go&#x98CE;&#x683C;&#x534F;&#x7A0B;&#x5E93;libmill](../.gitbook/assets/image%20%282%29.png)

### 实现的区别

虽然二者api比较一致，但是在实现上还是有较大区别，所以这里说“libmill是goroutine\*\***风格\*\***的协程库”。在api风格上很相似，那么它们有哪些不同点呢？

* 在libmill里面所有的协程调度都是在当前线程中的。也就是说一个单线程程序使用了libmill实现的协程，并且协程执行过程中使用了阻塞的系统调用，这样会阻塞整个进程。
* go中创建的协程会被分摊到多个物理线程上去执行，一个协程引起的线程阻塞不影响其他协程调度。一个协程中使用了阻塞的系统调用只会阻塞当前线程，并不会阻塞进程中的其他线程运行，就是说创建出来的其它协程仍可得到调度。

协程最终还是由线程来执行的，那也顺便提下Linux下线程库的实现，Linux下线程库有两个，比较早的是**LinuxThreads线程库**，现在用的一般都是**Native POSIX Threads Library（nptl）**，也就是pthread线程库。

* LinuxThreads是用户级线程库，创建的线程内核无感知，调度也是用户态线程调度器自己实现的；
* pthread线程库创建的线程都是一个LWP进程，它使用sys\_clone\(\)并传递CLONE\_THREAD选项来创建一个线程（本质上还是LWP）并且线程所属进程id相同。

### 工程结构

下面是libmill源码的工程结构，后面源码分析阶段我们将阅读并分析libmill的设计实现。

| 目录结构 | 描述 |
| :--- | :--- |
| ==作者、版权、配置、构建脚本== |  |
| ├── AUTHORS | 作者信息 |
| ├── README.md | 工程介绍 |
| ├── abi\_version.sh | [ABI](https://en.wikipedia.org/wiki/ABI)版本信息 |
| ├── package\_version.sh | 工程版本信息 |
| ├── autogen.sh | 自动化构建脚本 |
| ├── ... | ... |
| ==核心代码== |  |
| ├── libmill.h | libmill头文件 |
| ├── debug.c/.h | 调试函数 |
| ├── chan.c/.h | channel |
| ├── cr.c/.h | 协程coroutine |
| ├── file.c | file |
| ├── ip.c/.h | ip |
| ├── poller.c/.h ├── kqueue.inc/epoll.inc/poll.inc | io多路复用 |
| ├── mfork.c | 协程创建 |
| ├── tcp.c/udp.c/unix.c | tcp、udp、unix套接字 |
| ├── timer.h/.c | 定时器 |
| ├── utils.h | 常用工具方法 |
| ├── ssl.c | ssl支持 |
| ├── dns/ | dns解析 |
| ==链表、栈等数据结构== |  |
| ├── slist.h/.c | 单向链表 |
| ├── list.h/.c | 双向链表 |
| ├── stack.h/.c | 栈 |
| ==以下为测试代码== |  |
| ├── tests/ | 工程测试代码 |
| ├── perf/ | 工程性能测试代码 |
| ├── tutorial/ | 工程示例代码 |

要研习libmill代码也比较容易上手，比如对于`go(foo(arg1,arg2,arg3))`是启动一个协程并执行foo函数，想了解这里的协程是如何创建、执行、退出的，只需要从`go`这个函数实现层层展开就可以了。而相比较之下，在go语言中go只是一个关键字，要了解go关键字的功能实现在哪里（比如哪个函数）还是需要翻很多代码的。

关于libmill的简介，我们先简单提这些，下面再慢慢展开。

