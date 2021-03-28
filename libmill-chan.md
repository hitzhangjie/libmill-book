---
description: >-
  libmill是一个基于c语言开发的go风格协程库，实现了go风格的协程操作、chan操作、非阻塞的网络io操作等等，是一个不错的linux平台下c风格协程库实现。如果你想从0到1的快速了解如何开发一个协程库，libmill将是一个非常不错的案例；如果你想更深入地了解go，libmill里面也借鉴了go的一些设计思想；或者你想在生产环境中使用，开发者也提供了一个更健壮的版本libdill。
---

# chan: 通过通信来共享

## 设计思想

通信串行处理（CSP）和核心思想是，“**通过通信来共享数据，而非通过共享数据来通信**”，这是也是指导go语言chan的设计思想，libmill中也基于这一思想实现了chan。chan可以理解为管道，当有多个并发任务实体要共享数据时，通过这个chan将数据传递给对方，而非通过共享存储器数据的方式。

采用共享存储器数据的方式，也是一种思路，但是就要涉及到一些同步互斥的处理，这给并发编程带来了不小的负担，经常会有功能正确性问题、性能差问题出现，开发效率也低。

现在通过CSP的方式，我们将要共享的数据以值的形式通过chan发送给对方，接收到这个值的协程变为这个值的owner，再继续进行后续处理。思路清晰、编码简单、性能也好。

ps：至于为什么性能好，需要了解下进程、线程实现同步互斥所依赖的futex具体工作原理，也需要了解下chan的实现原理，以go为例，chan内部有锁保护队列，但是这里的锁也是基于sync.Mutex实现的，不是基于futex的。

## 源码实现

**file: chan.h**

```c
// choose语句是根据chan的状态来决定是否执行对应动作的分支控制语句
//
// 每个协程都会有一个choose数据结构来跟踪其当前正在执行的choose操作
struct mill_choosedata {
    // 每个choose语句中，又包含了多个从句构成的列表
    struct mill_slist clauses;
    // choose语句中otherwise从句是可选的，是否有otherwise从句，0否1是
    int othws;
    // 当前choose语句中，是否有指定deadline，未指定时为-1
    int64_t ddline;
    // 当前choose语句中，chan上事件就绪的从句数量
    int available;
};

// chan ep是对chan的使用者的描述，每个ep要么利用chan发送消息，要么接收消息
//
// 每个chan有一个sender和receiver，所以每个chan包括了sender、receiver两个mill_ep成员
struct mill_ep {
    // 类型（数据发送方 或 数据接收方）
    enum {MILL_SENDER, MILL_RECEIVER} type;
    // 初始化的choose操作的序号
    int seqnum;
    // choose语句中引用该mill_ep的从句数量
    int refs;
    // choose语句中引用该mill_ep并且已经处理过的数量
    int tmp;
    // choose语句中仍然在等待该mill_ep上事件就绪的从句列表
    struct mill_list clauses;
};

// chan
struct mill_chan_ {
    // channel里面存储的元素的尺寸(单位字节)
    size_t sz;
    // 每个chan上有一个seader和receiver
    // sender记录了等待在chan上执行数据发送操作的从句列表，receiver则记录了等待接收数据的从句列表
    struct mill_ep sender;
    struct mill_ep receiver;
    // 当前chan的引用计数（引用计数为0的时候chclose才会真正释放资源）
    int refcount;
    // 该chan上是否已经调用了chdone()，0否1是
    int done;
    // 存储消息数据的缓冲区紧跟在chan结构体后面
    // - bufsz代表消息缓冲区可容纳的最大消息数量
    // - items表示缓冲区中当前的消息数量
    // - first代表缓冲区中可接收的下一个消息的位置，缓冲区末尾有一个元素来存储chdone()写的数据
    size_t bufsz;
    size_t items;
    size_t first;
    // 调试信息
    struct mill_debug_chan debug;
};

// 该结构体代表choose语句中的一个从句，例如in、out、otherwise
struct mill_clause {
    // 等待this.ep事件就绪的从句列表(迭代器）
    struct mill_list_item epitem;
    // 该从句隶属的choose语句所包含的从句列表(迭代器)
    struct mill_slist_item chitem;
    // 创建该从句的协程
    struct mill_cr *cr;
    // 该从句正在等待的chan endpoint
    struct mill_ep *ep;
    // 对于out从句，val指向要发送的数据；对于in从句，val为NULL
    void *val;
    // 该从句执行完成后要跳转到第idx个从句
    int idx;
    // 是否有与当前从句匹配的pee(比如当前从句为ch上的写，是否有ch上的读从句)，0否1是
    int available;
    // 该从句是否在chan的sender或receiver列表中，0否1是
    int used;
};

// 返回包含该endpoint的chan
struct mill_chan_ *mill_getchan(struct mill_ep *ep);
```

**file: chan.c**

```c
// 每个choose语句都要分配一个单独的序号
static int mill_choose_seqnum = 0;
```

```c
// 返回包含ep的chan(根据端点类型获取)
struct mill_chan_ *mill_getchan(struct mill_ep *ep) {
    switch(ep->type) {
    case MILL_SENDER:
        return mill_cont(ep, struct mill_chan_, sender);
    case MILL_RECEIVER:
        return mill_cont(ep, struct mill_chan_, receiver);
    default:
        assert(0);
    }
}
```

```c
// 创建一个chan
struct mill_chan_ *mill_chmake_(size_t sz, size_t bufsz, const char *created) {
    mill_preserve_debug();
    // 分配消息缓冲区的时候多申请一个元素空间用于存chdone()提交的数据，
    // chdone不能写消息缓冲区，因为会因为缓冲区满而阻塞chdone()操作，
    // libmill是单线程调度，一个阻塞就会导致整个进程被阻塞了
    struct mill_chan_ *ch = 
        (struct mill_chan_*)malloc(sizeof(struct mill_chan_) + (sz * (bufsz + 1)));
    if(!ch)
        return NULL;
    mill_register_chan(&ch->debug, created);
    // 初始化chan
    ch->sz = sz;
    ch->sender.type = MILL_SENDER;
    ch->sender.seqnum = mill_choose_seqnum;
    mill_list_init(&ch->sender.clauses);
    ch->receiver.type = MILL_RECEIVER;
    ch->receiver.seqnum = mill_choose_seqnum;
    mill_list_init(&ch->receiver.clauses);
    ch->refcount = 1;
    ch->done = 0;
    ch->bufsz = bufsz;
    ch->items = 0;
    ch->first = 0;
    mill_trace(created, "<%d>=chmake(%d)", (int)ch->debug.id, (int)bufsz);
    return ch;
}
```

```c
// dup操作，只是增加chan引用计数
struct mill_chan_ *mill_chdup_(struct mill_chan_ *ch, const char *current) {
    if(mill_slow(!ch))
        mill_panic("null channel used");
    mill_trace(current, "chdup(<%d>)", (int)ch->debug.id);
    ++ch->refcount;
    return ch;
}
```

```c
// 关闭chan，实际上减少引用计数直到为0再释放chan
void mill_chclose_(struct mill_chan_ *ch, const char *current) {
    if(mill_slow(!ch))
        mill_panic("null channel used");
    mill_trace(current, "chclose(<%d>)", (int)ch->debug.id);
    assert(ch->refcount > 0);
    --ch->refcount;
    if(ch->refcount)
        return;
    // 仍有依赖该chan的从句存在的话，关闭chan会出错
    if(!mill_list_empty(&ch->sender.clauses) ||
          !mill_list_empty(&ch->receiver.clauses))
        mill_panic("attempt to close a channel while it is still being used");
    mill_unregister_chan(&ch->debug);
    // 释放chan
    free(ch);
}
```

```c
// 唤醒一个因为调用mill_choose_wait而阻塞的协程
// 
// choose从句中协程因为等待io事件而阻塞，所以这里唤醒阻塞的协程也意味着要清除掉这里的从句
static void mill_choose_unblock(struct mill_clause *cl) {
    struct mill_slist_item *it;
    struct mill_clause *itcl;
    for(it = mill_slist_begin(&cl->cr->choosedata.clauses); it; it = mill_slist_next(it)) {
        itcl = mill_cont(it, struct mill_clause, chitem);
        // 如果当前从句不再当前chan的sender/receiver列表中则不予处理；
        // 已经在的话则要将该从句删除，正式因为这个从句的io事件使得协程被阻塞的
        if(!itcl->used)
            continue;
        mill_list_erase(&itcl->ep->clauses, &itcl->epitem);
    }
    // 如果有指定deadline，也删除对应的定时器
    if(cl->cr->choosedata.ddline >= 0)
        mill_timer_rm(&cl->cr->timer);
    // 恢复该协程的执行
    mill_resume(cl->cr, cl->idx);
}
```

```c
// choose语句初始化
static void mill_choose_init(const char *current) {
    mill_set_current(&mill_running->debug, current);
    mill_slist_init(&mill_running->choosedata.clauses);
    mill_running->choosedata.othws = 0;
    mill_running->choosedata.ddline = -1;
    mill_running->choosedata.available = 0;
    ++mill_choose_seqnum;
}

void mill_choose_init_(const char *current) {
    mill_trace(current, "choose()");
    mill_running->state = MILL_CHOOSE;
    mill_choose_init(current);
}
```

```c
// choose in从句
void mill_choose_in_(void *clause, struct mill_chan_ *ch, size_t sz, int idx) {
    if(mill_slow(!ch))
        mill_panic("null channel used");
    if(mill_slow(ch->sz != sz))
        mill_panic("receive of a type not matching the channel");
    // 检查当前从句对应的可读事件是否就绪，就绪则++available记录一下
    int available = ch->done || !mill_list_empty(&ch->sender.clauses) || ch->items ? 1 : 0;
    if(available)
        ++mill_running->choosedata.available;
    // 如果当前从句可读事件未就绪，但是当前运行协程中choose语句中有从句事件就绪，返回
    if(!available && mill_running->choosedata.available)
        return;
    /* Fill in the clause entry. */
    struct mill_clause *cl = (struct mill_clause*) clause;
    cl->cr = mill_running;
    cl->ep = &ch->receiver;
    cl->val = NULL;
    cl->idx = idx;
    cl->available = available;
    cl->used = 1;
    mill_slist_push_back(&mill_running->choosedata.clauses, &cl->chitem);
    if(cl->ep->seqnum == mill_choose_seqnum) {
        ++cl->ep->refs;
        return;
    }
    cl->ep->seqnum = mill_choose_seqnum;
    cl->ep->refs = 1;
    cl->ep->tmp = -1;
}

// choose out从句
void mill_choose_out_(void *clause, struct mill_chan_ *ch, void *val, size_t sz, int idx) {
    if(mill_slow(!ch))
        mill_panic("null channel used");
    // 调用了chdone的chan不能再执行写操作
    if(mill_slow(ch->done))
        mill_panic("send to done-with channel");
    if(mill_slow(ch->sz != sz))
        mill_panic("send of a type not matching the channel");
    // 检查chan上是否写就绪
    int available = !mill_list_empty(&ch->receiver.clauses) || ch->items < ch->bufsz ? 1 : 0;
    if(available)
        ++mill_running->choosedata.available;
    // 如果chan上没有写就绪事件，但是当前协程上有其他choose从句事件就绪，返回
    if(!available && mill_running->choosedata.available)
        return;
    /* Fill in the clause entry. */
    struct mill_clause *cl = (struct mill_clause*) clause;
    cl->cr = mill_running;
    cl->ep = &ch->sender;
    cl->val = val;
    cl->available = available;
    cl->idx = idx;
    cl->used = 1;
    mill_slist_push_back(&mill_running->choosedata.clauses, &cl->chitem);
    if(cl->ep->seqnum == mill_choose_seqnum) {
        ++cl->ep->refs;
        return;
    }
    cl->ep->seqnum = mill_choose_seqnum;
    cl->ep->refs = 1;
    cl->ep->tmp = -1;
}
```

```c
// choose从句deadline对应的超时回调，销毁所有的choose从句并resume协程
static void mill_choose_callback(struct mill_timer *timer) {
    struct mill_cr *cr = mill_cont(timer, struct mill_cr, timer);
    struct mill_slist_item *it;
    for(it = mill_slist_begin(&cr->choosedata.clauses); it; it = mill_slist_next(it)) {
        struct mill_clause *itcl = mill_cont(it, struct mill_clause, chitem);
        mill_assert(itcl->used);
        mill_list_erase(&itcl->ep->clauses, &itcl->epitem);
    }
    mill_resume(cr, -1);
}

// choose deadline从句
void mill_choose_deadline_(int64_t ddline) {
    if(mill_slow(mill_running->choosedata.othws || mill_running->choosedata.ddline >= 0))
        mill_panic("multiple 'otherwise' or 'deadline' clauses in a choose statement");
    if(ddline < 0)
        return;
    mill_running->choosedata.ddline = ddline;
}

// choose otherwise从句
void mill_choose_otherwise_(void) {
    if(mill_slow(mill_running->choosedata.othws ||
          mill_running->choosedata.ddline >= 0))
        mill_panic("multiple 'otherwise' or 'deadline' clauses in a choose statement");
    mill_running->choosedata.othws = 1;
}

// 往chan追加数据val
static void mill_enqueue(struct mill_chan_ *ch, void *val) {
    // 如果chan上还有关联的receiver执行choose in从句，唤醒对应的协程收数据（当然先写数据再唤醒）
    if(!mill_list_empty(&ch->receiver.clauses)) {
        mill_assert(ch->items == 0);
        struct mill_clause *cl = mill_cont(
            mill_list_begin(&ch->receiver.clauses), struct mill_clause, epitem);
        // 写数据
        memcpy(mill_valbuf(cl->cr, ch->sz), val, ch->sz);
        // 唤醒收数据的协程
        mill_choose_unblock(cl);
        return;
    }
    // 只写数据
    assert(ch->items < ch->bufsz);
    size_t pos = (ch->first + ch->items) % ch->bufsz;
    memcpy(((char*)(ch + 1)) + (pos * ch->sz) , val, ch->sz);
    ++ch->items;
}

// 从chan中取队首的数据val
static void mill_dequeue(struct mill_chan_ *ch, void *val) {
    // 拿chan上sender的第一个choose out从句
    struct mill_clause *cl = mill_cont(
        mill_list_begin(&ch->sender.clauses), struct mill_clause, epitem);
    // chan中valbuf当前无数据可读
    if(!ch->items) {
        // 调用了chdone后肯定没有sender要发送数据了，直接拷走数据即可（chdone追加的）
        if(mill_slow(ch->done)) {
            mill_assert(!cl);
            memcpy(val, ((char*)(ch + 1)) + (ch->bufsz * ch->sz), ch->sz);
            return;
        }
        // 还没有调用chdone，直接从choose out从句中拷走数据，再唤醒因为执行choose out阻塞的协程
        mill_assert(cl);
        memcpy(val, cl->val, ch->sz);
        mill_choose_unblock(cl);
        return;
    }
    // chan中valbuf当前有数据可读
    // - 读取chan中的数据；
    // - 如果对应的choose out从句cl存在，则拷贝其数据到chan valbuf并唤醒执行该从句的协程
    memcpy(val, ((char*)(ch + 1)) + (ch->first * ch->sz), ch->sz);
    ch->first = (ch->first + 1) % ch->bufsz;
    --ch->items;
    if(cl) {
        assert(ch->items < ch->bufsz);
        size_t pos = (ch->first + ch->items) % ch->bufsz;
        memcpy(((char*)(ch + 1)) + (pos * ch->sz) , cl->val, ch->sz);
        ++ch->items;
        mill_choose_unblock(cl);
    }
}

// choose wait从句
int mill_choose_wait_(void) {
    struct mill_choosedata *cd = &mill_running->choosedata;
    struct mill_slist_item *it;
    struct mill_clause *cl;

    // 每个协程都有一个对应的choosedata数据结构
    //    
    // 如果当前有就绪的choose in/out从句，则选择一个并执行
    if(cd->available > 0) {
        // 只有1个就绪的choose从句直接去检查el->ep->type就知道干什么了
        // 如果有多个就绪的choose从句，随机选择一个就绪的从句去执行
        int chosen = cd->available == 1 ? 0 : (int)(random() % (cd->available));

        for(it = mill_slist_begin(&cd->clauses); it; it = mill_slist_next(it)) {
            cl = mill_cont(it, struct mill_clause, chitem);
            if(!cl->available)
                continue;
            if(!chosen)
                break;
            --chosen;
        }
        struct mill_chan_ *ch = mill_getchan(cl->ep);
        // 根据choose从句类型决定是向chan发送数据，还是从chan读取数据
        if(cl->ep->type == MILL_SENDER)
            mill_enqueue(ch, cl->val);
        else
            mill_dequeue(ch, mill_valbuf(cl->cr, ch->sz));
        mill_resume(mill_running, cl->idx);
        return mill_suspend();
    }

    // 如果没有choose in/out从句事件就绪但是有otherwise从句，直接执行otherwise从句
    // - 这里实际上相当于将当前运行的协程重新加入调度队列，然后主动挂起当前协程
    if(cd->othws) {
        mill_resume(mill_running, -1);
        return mill_suspend();
    }

    // 如果指定了deadline从句，为其启动一个定时器，并绑定超时回调
    if(cd->ddline >= 0)
        mill_timer_add(&mill_running->timer, cd->ddline, mill_choose_callback);

    // 其他情况下，将当前协程和被查询的chan进行注册，等到直到有一个choose从句unblock
    for(it = mill_slist_begin(&cd->clauses); it; it = mill_slist_next(it)) {
        cl = mill_cont(it, struct mill_clause, chitem);
        if(mill_slow(cl->ep->refs > 1)) {
            if(cl->ep->tmp == -1)
                cl->ep->tmp =
                    cl->ep->refs == 1 ? 0 : (int)(random() % cl->ep->refs);
            if(cl->ep->tmp) {
                --cl->ep->tmp;
                cl->used = 0;
                continue;
            }
            cl->ep->tmp = -2;
        }
        mill_list_insert(&cl->ep->clauses, &cl->epitem, NULL);
    }
    // 如果有多个协程并发的执行chdone，只可能有一个执行成功，其他的都必须阻塞在下面这行
    return mill_suspend();
}

// 获取正在运行的协程的chan数据存储缓冲区valbuf
void *mill_choose_val_(size_t sz) {
    return mill_valbuf(mill_running, sz);
}

// 向chan中发送数据
void mill_chs_(struct mill_chan_ *ch, void *val, size_t sz,
      const char *current) {
    if(mill_slow(!ch))
        mill_panic("null channel used");
    mill_trace(current, "chs(<%d>)", (int)ch->debug.id);
    mill_choose_init(current);
    mill_running->state = MILL_CHS;
    struct mill_clause cl;
    mill_choose_out_(&cl, ch, val, sz, 0);
    mill_choose_wait_();
}

// 从chan中接收数据
void *mill_chr_(struct mill_chan_ *ch, size_t sz, const char *current) {
    if(mill_slow(!ch))
        mill_panic("null channel used");
    mill_trace(current, "chr(<%d>)", (int)ch->debug.id);
    mill_running->state = MILL_CHR;
    mill_choose_init(current);
    struct mill_clause cl;
    mill_choose_in_(&cl, ch, sz, 0);
    mill_choose_wait_();
    return mill_choose_val_(sz);
}

// chan上的chdone操作
void mill_chdone_(struct mill_chan_ *ch, void *val, size_t sz,
      const char *current) {
    if(mill_slow(!ch))
        mill_panic("null channel used");
    mill_trace(current, "chdone(<%d>)", (int)ch->debug.id);
    if(mill_slow(ch->done))
        mill_panic("chdone on already done-with channel");
    if(mill_slow(ch->sz != sz))
        mill_panic("send of a type not matching the channel");
    /* Panic if there are other senders on the same channel. */
    if(mill_slow(!mill_list_empty(&ch->sender.clauses)))
        mill_panic("send to done-with channel");
    /* Put the channel into done-with mode. */
    ch->done = 1;

    // 在valbuf末尾再追加一个元素，不能chs往valbuf中写因为这样没有receiver的情况下会阻塞
    memcpy(((char*)(ch + 1)) + (ch->bufsz * ch->sz) , val, ch->sz);

    // 追加上述一个多余的元素后，需要唤醒chan上所有等待的receiver
    while(!mill_list_empty(&ch->receiver.clauses)) {
        struct mill_clause *cl = mill_cont(
            mill_list_begin(&ch->receiver.clauses), struct mill_clause, epitem);
        memcpy(mill_valbuf(cl->cr, ch->sz), val, ch->sz);
        mill_choose_unblock(cl);
    }
}
```

