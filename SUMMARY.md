# Table of contents

* [序言](README.md)
* [作者](author.md)

## Part I - 系统基础 <a id="ji-chu"></a>

* [服务编程模型](ji-chu/server-programming-model.md)
* [进程, 线程, 协程](ji-chu/process-thread-coroutine.md)
* [libmill协程库](ji-chu/libmill-intro.md)
* [libdill协程库](ji-chu/libdill-intro.md)
* [栈空间分配](ji-chu/stack-memory.md)
* [上下文切换](ji-chu/context-switching.md)
* [上下文切换开销](ji-chu/context-switching-cost.md)
* [线程模型](ji-chu/thread-model.md)

## Part II - 源码分析

## chan：通过通信来共享

---

* [chan: 通过通信来共享](share-by-communication.md)

## coroutine: 高效轻量级并发

---

* [coroutine: libmill.h](coroutine-libmill.md)
* [coroutine: cr.h/cr.c](coroutine-channel.md)
* [coroutine: mfork](coroutine-mfork.md)
* [coroutine: stack](coroutine-stack.md)

## 网络IO: 多路复用Hook系统调用

* [network: ip](wang-luo-io-duo-lu-fu-yong-hook-xi-tong-tiao-yong/network-ip.md)
* [network: tcp](wang-luo-io-duo-lu-fu-yong-hook-xi-tong-tiao-yong/network-tcp.md)
* [network: udp](wang-luo-io-duo-lu-fu-yong-hook-xi-tong-tiao-yong/network-udp.md)
* [network: unix](wang-luo-io-duo-lu-fu-yong-hook-xi-tong-tiao-yong/network-unix.md)
* [network: file](wang-luo-io-duo-lu-fu-yong-hook-xi-tong-tiao-yong/network-file.md)
* [network: poller](wang-luo-io-duo-lu-fu-yong-hook-xi-tong-tiao-yong/network-poller.md)

## 常用数据结构

* [通用链表及迭代器实现](chang-yong-shu-ju-jie-gou/datatype-iterator.md)
* [双向链表: list](chang-yong-shu-ju-jie-gou/datatype-list.md)
* [单向链表: slist](chang-yong-shu-ju-jie-gou/datatype-slist.md)

## 高精度定时器

* [定时器: timer](gao-jing-du-ding-shi-qi/datatype-timer.md)

## 调试说明

* [如何调试libmill](tiao-shi-shuo-ming/libmill-debug.md)

## 常用辅助方法

* [常用工具函数：函数+宏](chang-yong-fu-zhu-fang-fa/libmill-utils.md)

