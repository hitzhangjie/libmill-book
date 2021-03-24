---
description: >-
  libmill是一个基于c语言开发的go风格协程库，实现了go风格的协程操作、chan操作、非阻塞的网络io操作等等，是一个不错的linux平台下c风格协程库实现。如果你想从0到1的快速了解如何开发一个协程库，libmill将是一个非常不错的案例；如果你想更深入地了解go，libmill里面也借鉴了go的一些设计思想；或者你想在生产环境中使用，开发者也提供了一个更健壮的版本libdill。
---

# 定时器: timer

**1 timer.h**

```c
struct mill_timer {
    // mill_list_item结合mill_cont来实现了list iterator
    /* Item in the global list of all timers. */
    struct mill_list_item item;
    /* The deadline when the timer expires. -1 if the timer is not active. */
    int64_t expiry;
    /* Callback invoked when timer expires. Pfui Teufel! */
    mill_timer_callback callback;
};
```

```c
/* Test wheather the timer is active. */
#define mill_timer_enabled(tm)  ((tm)->expiry >= 0)

/* Add a timer for the running coroutine. */
void mill_timer_add(struct mill_timer *timer, int64_t deadline, mill_timer_callback callback);

/* Remove the timer associated with the running coroutine. */
void mill_timer_rm(struct mill_timer *timer);

/* Number of milliseconds till the next timer expires. If there are no timers returns -1. */
int mill_timer_next(void);

/* Resumes all coroutines whose timers have already expired. Returns zero if no coroutine was resumed, 1 otherwise. */
int mill_timer_fire(void);

/* Called after fork in the child process to deactivate all the timers inherited from the parent. */
void mill_timer_postfork(void);
```

**2 timer.c**

```c
// 定时器精度控制，rdtsc先后读取ticks差值超过这个值则gettimeofday更新last_now
#define MILL_CLOCK_PRECISION 1000000
```

```c
// 返回gettimeofday获取的系统时间，单位seconds
static int64_t mill_os_time(void) {
#if defined __APPLE__
    ...
#else
    struct timeval tv;
    int rc = gettimeofday(&tv, NULL);
    assert(rc == 0);
    return ((int64_t)tv.tv_sec) * 1000 + (((int64_t)tv.tv_usec) / 1000);
#endif
}
```

```c
// 获取当前系统时间（注意这里的时间是有cache的）
int64_t mill_now_(void) {
#if (defined __GNUC__ || defined __clang__) && (defined __i386__ || defined __x86_64__)
    // rdtsc获取系统启动后经历的cpu时钟周期数量
    uint32_t low;
    uint32_t high;
    __asm__ volatile("rdtsc" : "=a" (low), "=d" (high));

    int64_t tsc = (int64_t)((uint64_t)high << 32 | low);

    static int64_t last_tsc = -1;
    static int64_t last_now = -1;
    if(mill_slow(last_tsc < 0)) {
        last_tsc = tsc;
        last_now = mill_os_time();
    }
    // 如果在精度范围内返回上次获取的系统时间，超出精度范围则更新系统时间
    if(mill_fast(tsc - last_tsc <= (MILL_CLOCK_PRECISION / 2) && tsc >= last_tsc))
        return last_now;

    last_tsc = tsc;
    last_now = mill_os_time();
    return last_now;
#else
    return mill_os_time();
#endif
}
```

```c
// 定时器列表，列表中定时器是有序的，时间靠前的排列在前面
static struct mill_list mill_timers = {0};

// 往定时器列表中添加定时器，保证链表中定时器有序（按过期时间升序排列）
void mill_timer_add(struct mill_timer *timer, int64_t deadline, mill_timer_callback callback) {
    mill_assert(deadline >= 0);
    timer->expiry = deadline;
    timer->callback = callback;
    /* Move the timer into the right place in the ordered list
       of existing timers. TODO: This is an O(n) operation! */
    struct mill_list_item *it = mill_list_begin(&mill_timers);
    while(it) {
        struct mill_timer *tm = mill_cont(it, struct mill_timer, item);
        /* If multiple timers expire at the same momemt they will be fired
           in the order they were created in (> rather than >=). */
        if(tm->expiry > timer->expiry)
            break;
        it = mill_list_next(it);
    }
    mill_list_insert(&mill_timers, &timer->item, it);
}

// 从定时器列表中移除定时器
void mill_timer_rm(struct mill_timer *timer) {
    mill_assert(timer->expiry >= 0);
    mill_list_erase(&mill_timers, &timer->item);
    timer->expiry = -1;
}

// 返回定时器列表中首个定时器的剩余超时时间
int mill_timer_next(void) {
    if(mill_list_empty(&mill_timers))
        return -1;
    int64_t nw = now();
    int64_t expiry = mill_cont(mill_list_begin(&mill_timers), struct mill_timer, item) -> expiry;
    return (int) (nw >= expiry ? 0 : expiry - nw);
}

// 执行所有超时的定时器绑定的回调函数，返回是否有调用定时器的回调方法
int mill_timer_fire(void) {
    /* Avoid getting current time if there are no timers anyway. */
    if(mill_list_empty(&mill_timers))
        return 0;
    int64_t nw = now();
    int fired = 0;
    while(!mill_list_empty(&mill_timers)) {
        struct mill_timer *tm = mill_cont(
            mill_list_begin(&mill_timers), struct mill_timer, item);
        if(tm->expiry > nw)
            break;
        mill_list_erase(&mill_timers, mill_list_begin(&mill_timers));
        tm->expiry = -1;
        if(tm->callback)
            tm->callback(tm);
        fired = 1;
    }
    return fired;
}

// 初始化定时器列表（postfork？fixme!!!）
void mill_timer_postfork(void) {
    mill_list_init(&mill_timers);
}
```

