---
description: >-
  libmill是一个基于c语言开发的go风格协程库，实现了go风格的协程操作、chan操作、非阻塞的网络io操作等等，是一个不错的linux平台下c风格协程库实现。如果你想从0到1的快速了解如何开发一个协程库，libmill将是一个非常不错的案例；如果你想更深入地了解go，libmill里面也借鉴了go的一些设计思想；或者你想在生产环境中使用，开发者也提供了一个更健壮的版本libdill。
---

# network: poller



#### poller

libmill里面网络io是基于io多路复用实现的，在linux平台下面基于epoll实现，在其他平台下面基于对应的io多路复用系统调用实现，这里只关注linux平台下的实现细节。

**poller.h**

```c
void mill_poller_init(void);

// poller也实现了libmill.h中定义的mill_wait()和mill_fdwait()

// 等待至少一个coroutine恢复执行
// 如果block=0，轮询事件时没有事件就绪则立即返回
// 如果block=1，轮询事件时没有事件就绪则阻塞，直到至少有一个事件就绪
void mill_wait(int block);

// 子进程中调用该函数将创建一个全新的pollset，与父进程的pollset脱离关系
void mill_poller_postfork(void);
```

**poller.c**

```c
/* Forward declarations for the functions implemented by specific poller
   mechanisms (poll, epoll, kqueue). */
void mill_poller_init(void);
static void mill_poller_add(int fd, int events);
static void mill_poller_rm(struct mill_cr *cr);
static void mill_poller_clean(int fd);
static int mill_poller_wait(int timeout);

// poller是否已经被初始化过，1是，0否
static int mill_poller_initialised = 0;

// 检查poller是否已初始化，没有初始化则初始化，已经初始化不处理
#define check_poller_initialised() \
do {\
    if(mill_slow(!mill_poller_initialised)) {\
        mill_poller_init();\
        mill_assert(errno == 0);\
        mill_main.fd = -1;\
        mill_main.timer.expiry = -1;\
        mill_poller_initialised = 1;\
    }\
} while(0)

// 挂起当前运行中的coroutine一定的时间
void mill_msleep_(int64_t deadline, const char *current) {
    mill_fdwait_(-1, 0, deadline, current);
}

// 
static void mill_poller_callback(struct mill_timer *timer) {
    struct mill_cr *cr = mill_cont(timer, struct mill_cr, timer);
    mill_resume(cr, -1);
    if (cr->fd != -1)
        mill_poller_rm(cr);
}

// 等待fd上的events就绪或者直到超时
int mill_fdwait_(int fd, int events, int64_t deadline, const char *current) {
    check_poller_initialised();
    // 如果指定了deadline，则添加一个定时器，超时回调mill_poller_callback唤醒定时器关联的协程
    if(deadline >= 0)
        mill_timer_add(&mill_running->timer, deadline, mill_poller_callback);
    // 如果指定了fd，则注册coroutine对该fd上的io事件监听
    if(fd >= 0)
        mill_poller_add(fd, events);
    // 执行了实际的wait，mill_suspend挂起当前协程切换到其他ready状态的协程
    mill_running->state = fd < 0 ? MILL_MSLEEP : MILL_FDWAIT;
    mill_running->fd = fd;
    mill_running->events = events;
    mill_set_current(&mill_running->debug, current);
    // 当前协程要等待其他协程调用mill_resume来唤醒当前协程才会从这里继续向下执行
    // mill_suspend返回值为mill_resume(struct mill_cr *cr, int result)中的参数result
    // 谁来唤醒当前协程呢？epfd事件轮询的时候有fd事件就绪则会唤醒等待在该fd上的协程！
    int rc = mill_suspend();
    /* Handle file descriptor events. */
    if(rc >= 0) {
        mill_assert(!mill_timer_enabled(&mill_running->timer));
        return rc;
    }
    /* Handle the timeout. */
    mill_assert(mill_running->fd == -1);
    return 0;
}

// 将fd从epfd上取消注册
void mill_fdclean_(int fd) {
    check_poller_initialised();
    mill_poller_clean(fd);
}

// 事件轮询，block=1表示阻塞直到有事件到达（这里的事件包括定时器超时事件、io就绪事件） 
void mill_wait(int block) {
    check_poller_initialised();
    while(1) {
        // 计算下次轮询的超时时间
        int timeout = block ? mill_timer_next() : 0;
        // 检查fd上的io事件
        int fd_fired = mill_poller_wait(timeout);
        // 检查定时器超时时间
        int timer_fired = mill_timer_fire();
        // 非阻塞情况下不重试
        if(!block || fd_fired || timer_fired)
            break;
        /* If timeout was hit but there were no expired timers do the poll
           again. This should not happen in theory but let's be ready for the
           case when the system timers are not precise. */
    }
}

// libmill在linux平台下基于epoll实现事件轮询
#include "epoll.inc"
```

**epoll.inc**

```c
#define MILL_ENDLIST 0xffffffff
#define MILL_EPOLLSETSIZE 128

// 全局pollset，linux下其实是个epfd
static int mill_efd = -1;

// epoll允许为每个fd注册一个单独的指针，我们对每个fd需要注册两个coroutine指针，
// 一个是等待从fd接收数据的coroutine指针，一个是等待向fd发送数据的coroutine指针
// 因此，我们需要创建一个coroutine指针pair数组来跟踪打开的socket fds上的io
struct mill_crpair {
    struct mill_cr *in;
    struct mill_cr *out;
    uint32_t currevs;
    // 1-based索引值，0代表不属于这个list，MILL_ENDLIST代表list结束（通过数组索引实现的链表）
    uint32_t next;
};

static struct mill_crpair *mill_crpairs = NULL;
static int mill_ncrpairs = 0;
static uint32_t mill_changelist = MILL_ENDLIST;

// poller初始化
void mill_poller_init(void) {
    // 检查系统允许的进程最大可打开文件数量
    struct rlimit rlim;
    int rc = getrlimit(RLIMIT_NOFILE, &rlim);
    if(mill_slow(rc < 0)) 
        return;
    mill_ncrpairs = rlim.rlim_max;
    // 最多可打开mill_ncrpairs个socket，每个socket应该有对应的一个struct_crpair
    mill_crpairs = (struct mill_crpair*)calloc(mill_ncrpairs, sizeof(struct mill_crpair));
    if(mill_slow(!mill_crpairs)) {
        errno = ENOMEM; 
        return;
    }
    // 创建epfd准备socket io事件进行监听
    mill_efd = epoll_create(1);
    if(mill_slow(mill_efd < 0)) {
        free(mill_crpairs);
        mill_crpairs = NULL;
        return;
    }
    errno = 0;
}

// 子进程中重新创建一个pollset，与父进程的pollset脱离了关系
void mill_poller_postfork(void) {
    if(mill_efd != -1) {
        int rc = close(mill_efd);
        mill_assert(rc == 0);
    }
    mill_efd = -1;
    mill_crpairs = NULL;
    mill_ncrpairs = 0;
    mill_changelist = MILL_ENDLIST;
    mill_poller_init();
}

// 注册当前协程对fd上events就绪的监听
static void mill_poller_add(int fd, int events) {
    // 每个fd都是从最小未被使用的整数开始分配的，因此fd取值一定在[0,rlim.rlim_max]范围内，
    // 因此，这里的mill_crpairs[fd]一定不会出现内存越界
    struct mill_crpair *crp = &mill_crpairs[fd];
    if(events & FDW_IN) {
        // 不允许多个协程wait同一个fd，不然io就乱了
        if(crp->in)
            mill_panic("multiple coroutines waiting for a single file descriptor");
        crp->in = mill_running;
    }
    if(events & FDW_OUT) {
        // 不允许多个协程wait同一个fd，不然io就乱了
        if(crp->out)
            mill_panic("multiple coroutines waiting for a single file descriptor");
        crp->out = mill_running;
    }
    if(!crp->next) {
        crp->next = mill_changelist;
        mill_changelist = fd + 1;           // 1-based索引值
    }
}

// 取消协程cr当前对fd上事件的监听
// 注意这里并没有从epfd中清除对fd的监听，其他协程还可能监听呢
static void mill_poller_rm(struct mill_cr *cr) {
    int fd = cr->fd;
    mill_assert(fd != -1);
    struct mill_crpair *crp = &mill_crpairs[fd];
    if(crp->in == cr) {
        crp->in = NULL;
        cr->fd = -1;
    }
    if(crp->out == cr) {
        crp->out = NULL;
        cr->fd = -1;
    }
    if(!crp->next) {
        crp->next = mill_changelist;
        mill_changelist = fd + 1;           // 1-based索引值
    }
}

// 从epfd中清除对fd的监听
// 注意必须所有coroutine都没有监听这个fd
static void mill_poller_clean(int fd) {
    struct mill_crpair *crp = &mill_crpairs[fd];
    // 断言必须所有coroutine都没有监听这个fd
    mill_assert(!crp->in);
    mill_assert(!crp->out);
    /* Remove the file descriptor from the pollset, if it is still present. */
    if(crp->currevs) {   
        struct epoll_event ev;
        ev.data.fd = fd;
        ev.events = 0;
        int rc = epoll_ctl(mill_efd, EPOLL_CTL_DEL, fd, &ev);
        mill_assert(rc == 0 || errno == ENOENT);
    }
    /* Clean the cache. */
    crp->currevs = 0;
    if(!crp->next) {
        crp->next = mill_changelist;
        mill_changelist = fd + 1;
    }
}

// epoll轮询事件就绪状态
static int mill_poller_wait(int timeout) {
    /* Apply any changes to the pollset.
       TODO: Use epoll_ctl_batch once available. */
    while(mill_changelist != MILL_ENDLIST) {
        int fd = mill_changelist - 1;
        struct mill_crpair *crp = &mill_crpairs[fd];
        struct epoll_event ev;
        ev.data.fd = fd;
        ev.events = 0;
        // 根据crp中是否有coroutine监听crp->fd上的io事件来更新crp->currevs，并更新epfd事件注册
        if(crp->in)
            ev.events |= EPOLLIN;
        if(crp->out)
            ev.events |= EPOLLOUT;
        if(crp->currevs != ev.events) {
            int op;
            if(!ev.events)
                 op = EPOLL_CTL_DEL;
            else if(!crp->currevs)
                 op = EPOLL_CTL_ADD;
            else
                 op = EPOLL_CTL_MOD;
            crp->currevs = ev.events;
            int rc = epoll_ctl(mill_efd, op, fd, &ev);
            mill_assert(rc == 0);
        }
        mill_changelist = crp->next;
        crp->next = 0;
    }
    // epoll_wait事件轮询，返回就绪事件
    struct epoll_event evs[MILL_EPOLLSETSIZE];
    int numevs;
    while(1) {
        numevs = epoll_wait(mill_efd, evs, MILL_EPOLLSETSIZE, timeout);
        if(numevs < 0 && errno == EINTR)
            continue;
        mill_assert(numevs >= 0);
        break;
    }
    // 遍历fd就绪的事件，并唤醒等待该fd事件就绪的coroutine
    int i;
    for(i = 0; i != numevs; ++i) {
        struct mill_crpair *crp = &mill_crpairs[evs[i].data.fd];
        int inevents = 0;
        int outevents = 0;
        /* Set the result values. */
        if(evs[i].events & EPOLLIN)
            inevents |= FDW_IN;
        if(evs[i].events & EPOLLOUT)
            outevents |= FDW_OUT;
        if(evs[i].events & (EPOLLERR | EPOLLHUP)) {
            inevents |= FDW_ERR;
            outevents |= FDW_ERR;
        }
        // 唤醒等待该fd上就绪事件的coroutine
        if(crp->in == crp->out) {
            struct mill_cr *cr = crp->in;
            mill_resume(cr, inevents | outevents);
            mill_poller_rm(cr);
            if(mill_timer_enabled(&cr->timer))
                mill_timer_rm(&cr->timer);
        }
        else {
            // 唤醒等待该fd读就绪事件的协程
            if(crp->in && inevents) {
                struct mill_cr *cr = crp->in;
                mill_resume(cr, inevents);
                mill_poller_rm(cr);
                if(mill_timer_enabled(&cr->timer))
                    mill_timer_rm(&cr->timer);
            }
            // 唤醒等待该fd写就绪事件的协程
            if(crp->out && outevents) {
                struct mill_cr *cr = crp->out;
                mill_resume(cr, outevents);
                mill_poller_rm(cr);
                if(mill_timer_enabled(&cr->timer))
                    mill_timer_rm(&cr->timer);
            }
        }
    }

    // 至少有一个协程被唤醒则返回1，反之返回0
    return numevs > 0 ? 1 : 0;
}
```

