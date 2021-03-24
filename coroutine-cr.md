---
description: >-
  libmill是一个基于c语言开发的go风格协程库，实现了go风格的协程操作、chan操作、非阻塞的网络io操作等等，是一个不错的linux平台下c风格协程库实现。如果你想从0到1的快速了解如何开发一个协程库，libmill将是一个非常不错的案例；如果你想更深入地了解go，libmill里面也借鉴了go的一些设计思想；或者你想在生产环境中使用，开发者也提供了一个更健壮的版本libdill。
---

# coroutine: cr.h/cr.c

**file: cr.h**

```c
// coroutine state
enum mill_state {
    MILL_READY,         //可以被调度
    MILL_MSLEEP,        //mill_suspend挂起等待mill_resume唤醒
    MILL_FDWAIT,        //mill_fdwait_，等待mill_poller_wait或者timer回调唤醒
    MILL_CHR,           //...
    MILL_CHS,           //...
    MILL_CHOOSE         //...
};

/* 
   协程内存布局如下：
   +----------------------------------------------------+--------+---------+
   |                                              stack | valbuf | mill_cr |
   +----------------------------------------------------+--------+---------+
   - mill_cr：包括coroutine的通用信息
   - valbuf：临时存储从chan中接收到的数据
   - stack：标准的c程序栈，栈从高地址向低地址方向增长
*/
struct mill_cr {
    // 协程状态，用于调试目的
    enum mill_state state;

    // 协程如果没有阻塞并且等待执行，会被加入到ready队列中，并设置is_ready=1；
    // 反之，设置is_ready=0，不加入ready队列中
    int is_ready;
    struct mill_slist_item ready;

    // 如果协程需要等待一个截止时间，就需要下面的定时器来实现超时回调
    struct mill_timer timer;

    // 协程在fdwait中等待fd上的io事件就绪，若fd为-1表示当前协程没关注特定fd上的io事件
    int fd;

    // 协程在fdwait中等待fd上的io就绪事件events，用于调试目的
    int events;

    // 协程执行choose语句时要使用的结构体
    struct mill_choosedata choosedata;

    // 协程暂停、恢复执行的时候需要保存、还原其上下文信息
#if defined(__x86_64__)
    uint64_t ctx[10];
#else
    sigjmp_buf ctx;
#endif

    // suspend挂起协程后resume恢复协程执行，resume第二个参数result会被设置到cr->result成员；
    // 其他协程suspend并切换到被resumed的线程时会return mill_running->result
    int result;

    // 如果协程需要的valbuf比预设的mill_valbuf要大的话，那就得从heap中动态分配；
    // 分配的内存空间地址、尺寸记录在这两个成员中
    void *valbuf;
    size_t valbuf_sz;

    // 协程本地存储（有点类似线程local存储）
    void *clsval;

#if defined MILL_VALGRIND
    /* Valgrind stack identifier. */
    int sid;
#endif

    // 调试信息
    struct mill_debug_cr debug;
};

// 主线程对应的假的coroutine
extern struct mill_cr mill_main;

// 记录当前正在运行的协程
extern struct mill_cr *mill_running;

// 挂起当前正在运行的协程，并切换到一个不同的is_ready=1的协程取运行；
// 一旦某个协程resume这个被挂起的协程，resume中传递的参数result将被该suspend函数返回
int mill_suspend(void);

// 调度之前被挂起的协程cr恢复执行，其实只是将其加入ready队列等待被调度而已
void mill_resume(struct mill_cr *cr, int result);

// 返回一个执行协程临时数据区valbuf的指针，返回的数据区容量至少为size bytes
void *mill_valbuf(struct mill_cr *cr, size_t size);

// 子进程中调用，目的是为了停止运行从父进程继承的协程
void mill_cr_postfork(void);
```

**file: cr.c**

```c
// 协程临时数据区valbuf的大小，这里的临时数据区应该合理对齐；
// 如果当前有任何分配的协程栈，就不应该改变这里的尺寸，可能会影响到协程不同内存区域的计算
size_t mill_valbuf_size = 128;

// 主线程这个假协程对应的valbuf
char mill_main_valbuf[128];

volatile int mill_unoptimisable1_ = 1;
volatile void *mill_unoptimisable2_ = NULL;

// 主协程
struct mill_cr mill_main = {0};

// 默认当前正在运行的协程就是mill_run
struct mill_cr *mill_running = &mill_main;

// 等待被调度的就绪协程队列
struct mill_slist mill_ready = {0};

// 返回当前上下文信息
inline mill_ctx mill_getctx_(void) {
#if defined __x86_64__
    return mill_running->ctx;
#else
    return &mill_running->ctx;
#endif
}

// 返回协程临时数据区valbuf的起始地址
static void *mill_getvalbuf(struct mill_cr *cr, size_t size) {
    // 如果请求较小的valbuf则不需要在heap上动态分配
    // 另外要注意主协程没有为其分配栈，但是单独为其分配了valbuf
    if(mill_fast(cr != &mill_main)) {
        if(mill_fast(size <= mill_valbuf_size))
            return (void*)(((char*)cr) - mill_valbuf_size);
    }
    else {
        if(mill_fast(size <= sizeof(mill_main_valbuf)))
            return (void*)mill_main_valbuf;
    }
    // 如果请求较大的valbuf则需要在heap上动态分配，fixme!!!
    if(mill_fast(cr->valbuf && cr->valbuf_sz <= size))
        return cr->valbuf;
    void *ptr = realloc(cr->valbuf, size);
    if(!ptr)
        return NULL;
    cr->valbuf = ptr;
    cr->valbuf_sz = size;
    return cr->valbuf;
}

// 预准备count个协程，并分别初始化其栈尺寸、valbuf、valbuf_sz
void mill_goprepare_(int count, size_t stack_size, size_t val_size) {
    if(mill_slow(mill_hascrs())) {errno = EAGAIN; return;}
    // poller初始化
    mill_poller_init();
    if(mill_slow(errno != 0)) return;
    // 可能的话尅设置val_size稍微大一点以便能合理内存对齐
    mill_valbuf_size = (val_size + 15) & ~((size_t)0xf);
    // 为主协程（假的）分配valbuf
    if(mill_slow(!mill_getvalbuf(&mill_main, mill_valbuf_size))) {
        errno = ENOMEM;
        return;
    }
    // 为协程分配栈（这里分配时计算了stack+valbuf+mill_cr，是一个完整协程的内存空间大小）
    mill_preparestacks(count, stack_size + mill_valbuf_size + sizeof(struct mill_cr));
}

// 挂起当前正在运行的协程，并切换到一个is_ready=1的协程上去执行
// 被挂起的协程需要另一个协程调用resume(cr, result)方法来恢复其执行，恢复后suspend将返回result
int mill_suspend(void) {
    /* Even if process never gets idle, we have to process external events
       once in a while. The external signal may very well be a deadline or
       a user-issued command that cancels the CPU intensive operation. */
    static int counter = 0;
    if(counter >= 103) {
        mill_wait(0);
        counter = 0;
    }
    // 保存当前协程运行时的上下文信息
    if(mill_running) {
        mill_ctx ctx = mill_getctx_();
        if (mill_setjmp_(ctx))
            return mill_running->result;
    }
    while(1) {
        // 寻找一个is_ready=1的可运行的协程并恢复其执行
        if(!mill_slist_empty(&mill_ready)) {
            ++counter;
            struct mill_slist_item *it = mill_slist_pop(&mill_ready);
            mill_running = mill_cont(it, struct mill_cr, ready);
            mill_assert(mill_running->is_ready == 1);
            mill_running->is_ready = 0;
            mill_longjmp_(mill_getctx_());
        }
        // 找不到就要wait，可能要挂起当前协程直到被外部事件唤醒（io事件或者定时器超时）
        mill_wait(1);
        mill_assert(!mill_slist_empty(&mill_ready));
        counter = 0;
    }
}

// 恢复一个协程的运行，每个协程cr都在其内部保存了其运行时上下文信息
// 这里其实只是将其重新加入就绪队列等待被调度而已
inline void mill_resume(struct mill_cr *cr, int result) {
    mill_assert(!cr->is_ready);
    cr->result = result;
    cr->state = MILL_READY;
    cr->is_ready = 1;
    mill_slist_push_back(&mill_ready, &cr->ready);
}

/* mill_prologue_() and mill_epilogue_() live in the same scope with
   libdill's stack-switching black magic. As such, they are extremely
   fragile. Therefore, the optimiser is prohibited to touch them. */
#if defined __clang__
#define dill_noopt __attribute__((optnone))
#elif defined __GNUC__
#define dill_noopt __attribute__((optimize("O0")))
#else
#error "Unsupported compiler!"
#endif

// go()开始部分，启动一个新的协程，返回指向栈顶的指针
__attribute__((noinline)) dill_noopt 
void *mill_prologue_(const char *created) {
    ......
    // 分配并初始化新的stack
#if defined MILL_VALGRIND
    ......
#else
    // 先从cache中取，取不到动态分配
    struct mill_cr *cr = ((struct mill_cr*)mill_allocstack(NULL)) - 1;
#endif
    mill_register_cr(&cr->debug, created);
    cr->is_ready = 0;
    cr->valbuf = NULL;
    cr->valbuf_sz = 0;
    cr->clsval = NULL;
    cr->timer.expiry = -1;
    cr->fd = -1;
    cr->events = 0;
    mill_trace(created, "[%d]=go()", (int)cr->debug.id);
    // 挂起父协程并调度新创建的协程来运行
    mill_resume(mill_running, 0);    
    mill_running = cr;
    // 计算返回valbuf栈顶尺寸
    return (void*)(((char*)cr) - mill_valbuf_size);
}

// go结束部分，协程结束的时候执行清零动作
__attribute__((noinline)) dill_noopt
void mill_epilogue_(void) {
    mill_trace(NULL, "go() done");
    mill_unregister_cr(&mill_running->debug);
    if(mill_running->valbuf)
        free(mill_running->valbuf);
#if defined MILL_VALGRIND
    ......
#endif
    mill_freestack(mill_running + 1);
    mill_running = NULL;
    // 考虑到这里没有运行中的协程了，所以mill_suspend永远不会返回了
    mill_suspend();
}

void mill_yield_(const char *current) {
    mill_trace(current, "yield()");
    mill_set_current(&mill_running->debug, current);
    // 这里看起来有点可疑，但是没问题，我们可以在挂起一个协程之前就resume它来执行；
    // 这样做的目的是为了suspend之后能够使该协程重新获得被调度执行的机会
    mill_resume(mill_running, 0);
    mill_suspend();
}

// 返回valbuf起始地址
void *mill_valbuf(struct mill_cr *cr, size_t size) {
    void *ptr = mill_getvalbuf(cr, size);
    if(!ptr)
        mill_panic("not enough memory to receive from channel");
    return ptr;
}

// 返回协程本地存储指针
void *mill_cls_(void) {
    return mill_running->clsval;
}

// 设置协程本地存储操作
void mill_setcls_(void *val) {
    mill_running->clsval = val;
}

// fork之后子进程清空就绪协程队列列表
void mill_cr_postfork(void) {
    /* Drop all coroutines in the "ready to execute" list. */
    mill_slist_init(&mill_ready);
}
```

