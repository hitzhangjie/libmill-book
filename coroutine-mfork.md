---
description: >-
  libmill是一个基于c语言开发的go风格协程库，实现了go风格的协程操作、chan操作、非阻塞的网络io操作等等，是一个不错的linux平台下c风格协程库实现。如果你想从0到1的快速了解如何开发一个协程库，libmill将是一个非常不错的案例；如果你想更深入地了解go，libmill里面也借鉴了go的一些设计思想；或者你想在生产环境中使用，开发者也提供了一个更健壮的版本libdill。
---

# coroutine: mfork



#### mfork

```c
// 创建子进程
pid_t mill_mfork_(void) {
    pid_t pid = fork();
    if(pid != 0) {
        // 父进程
        return pid;
    }
    // 子进程，这里会对子进程进行一些特殊的处理
    // 包括重新初始化协程队列mill_ready、fd监听pollset、定时器timers list
    mill_cr_postfork();
    mill_poller_postfork();
    mill_timer_postfork();
    return 0;
}
```

