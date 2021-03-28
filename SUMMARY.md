# Table of contents

* [序言](README.md)
* [作者](author.md)

## Part I - 系统基础 <a id="basics"></a>

* [服务编程模型](basics/server-programming-model.md)
* [进程, 线程, 协程](basics/process-thread-coroutine.md)
* [栈空间分配](basics/stack-memory.md)
* [上下文切换](basics/context-switching.md)
* [上下文切换开销](basics/context-switching-cost.md)
* [线程模型](basics/thread-model.md)

## Part II - 源码分析 <a id="source"></a>

* [libmill协程库](source/libmill-intro.md)
* [libdill协程库](source/libdill-intro.md)

## chan：通过通信来共享 <a id="communication"></a>

---

* [chan: 通过通信来共享](libmill-chan.md)

## coroutine: 高效轻量级并发 <a id="coroutine"></a>

---

* [coroutine: libmill.h](coroutine-libmill.md)
* [coroutine: cr.h/cr.c](coroutine-cr.md)
* [coroutine: mfork](coroutine-mfork.md)
* [coroutine: stack](coroutine-stack.md)

## 网络IO: 多路复用Hook系统调用 <a id="io-multiplexing"></a>

* [network: ip](io-multiplexing/network-ip.md)
* [network: tcp](io-multiplexing/network-tcp.md)
* [network: udp](io-multiplexing/network-udp.md)
* [network: unix](io-multiplexing/network-unix.md)
* [network: file](io-multiplexing/network-file.md)
* [network: poller](io-multiplexing/network-poller.md)

## 常用数据结构 <a id="datatypes"></a>

* [通用链表及迭代器实现](datatypes/datatype-iterator.md)
* [双向链表: list](datatypes/datatype-list.md)
* [单向链表: slist](datatypes/datatype-slist.md)

## 高精度定时器 <a id="timer"></a>

* [定时器: timer](timer/libmill-timer.md)

## 调试说明 <a id="debugging"></a>

* [如何调试libmill](debugging/libmill-debugging.md)

## 常用辅助方法 <a id="utils"></a>

* [常用工具函数：函数+宏](utils/libmill-utils.md)

