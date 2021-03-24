# Table of contents

* [序言](README.md)
* [作者](author.md)

## Part I - 系统基础 <a id="ji-chu"></a>

* [服务编程模型](ji-chu/fu-wu-bian-cheng-mo-xing.md)
* [进程, 线程, 协程](ji-chu/libdill-jian-jie.md)
* [libmill协程库](ji-chu/1-jian-jie.md)
* [libdill协程库](ji-chu/libdill.md)
* [栈空间分配](ji-chu/zhan-kong-jian-fen-pei.md)
* [上下文切换](ji-chu/shang-xia-wen-qie-huan.md)
* [上下文切换开销](ji-chu/shang-xia-wen-qie-huan-kai-xiao.md)
* [线程模型](ji-chu/xian-cheng-mo-xing.md)

## Part II - 源码分析

## chan：通过通信来共享

---

* [chan: 通过通信来共享](2-tong-guo-tong-xin-lai-gong-xiang-chan-shi-xian.md)

## coroutine: 高效轻量级并发

---

* [coroutine: libmill.h](coroutine-libmill.h.md)
* [coroutine: cr.h/cr.c](3-gao-xiao-qing-liang-ji-bing-fa-coroutine.md)
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

* [通用链表及迭代器实现](chang-yong-shu-ju-jie-gou/tong-yong-lian-biao-ji-die-dai-qi-shi-xian.md)
* [双向链表: list](chang-yong-shu-ju-jie-gou/shuang-xiang-lian-biao-list.md)
* [单向链表: slist](chang-yong-shu-ju-jie-gou/dan-xiang-lian-biao-slist.md)

## 高精度定时器

* [定时器: timer](gao-jing-du-ding-shi-qi/ding-shi-qi-timer.md)

## 调试说明

* [如何调试libmill](tiao-shi-shuo-ming/untitled.md)

## 常用辅助方法

* [常用工具函数：函数+宏](chang-yong-fu-zhu-fang-fa/chang-yong-gong-ju-han-shu-han-shu-+-hong.md)

