---
description: >-
  libmill是一个基于c语言开发的go风格协程库，实现了go风格的协程操作、chan操作、非阻塞的网络io操作等等，是一个不错的linux平台下c风格协程库实现。如果你想从0到1的快速了解如何开发一个协程库，libmill将是一个非常不错的案例；如果你想更深入地了解go，libmill里面也借鉴了go的一些设计思想；或者你想在生产环境中使用，开发者也提供了一个更健壮的版本libdill。
---

# coroutine: stack



#### stack

stack是coroutine stack的抽象，这里coroutine stack可以看作是一个trick，我们把要并发执行的任务放在一个coroutine stack上执行，并且允许程序上下文在这些并发的任务之间来回切换，以实现更细粒度的并发。每个并发任务都有一个coroutine stack与之对应，每个任务中都涉及到对栈的操作，对栈的操作与普通程序对栈的操作一样都是从高地址向低地址方向增长的，这是由编译器决定的。

**file: stack.h**

```c
/* Purges all the existing cached stacks and preallocates 'count' new stacks
   of size 'stack_size'. Sets errno in case of error. */
void mill_preparestacks(int count, size_t stack_size);

/* Allocates new stack. Returns pointer to the *top* of the stack.
   For now we assume that the stack grows downwards. */
void *mill_allocstack(size_t *stack_size);

/* Deallocates a stack. The argument is pointer to the top of the stack. */
void mill_freestack(void *stack);
```

**file: stack.c**

```c
// 获取内存页面大小（只查询一次）
static size_t mill_page_size(void) {
    static long pgsz = 0;
    if(mill_fast(pgsz))
        return (size_t)pgsz;
    pgsz = sysconf(_SC_PAGE_SIZE);
    mill_assert(pgsz > 0);
    return (size_t)pgsz;
}

// stack size，也可以由用户指定
static size_t mill_stack_size = 256 * 1024 - 256;

// 实际的stack size
static size_t mill_sanitised_stack_size = 0;

// 获取stack size
static size_t mill_get_stack_size(void) {
#if defined HAVE_POSIX_MEMALIGN && HAVE_MPROTECT
    /* If sanitisation was already done, return the precomputed size. */
    if(mill_fast(mill_sanitised_stack_size))
        return mill_sanitised_stack_size;
    mill_assert(mill_stack_size > mill_page_size());
    /* Amount of memory allocated must be multiply of the page size otherwise
       the behaviour of posix_memalign() is undefined. */
    size_t sz = (mill_stack_size + mill_page_size() - 1) &
        ~(mill_page_size() - 1);
    /* Allocate one additional guard page. */
    mill_sanitised_stack_size = sz + mill_page_size();
    return mill_sanitised_stack_size;
#else
    return mill_stack_size;
#endif
}

// 未使用的cached的stack的最大数量
// 如果我们的代码还在一个stack上运行那就不能释放它，因此至少需要有一个cached的stack
static int mill_max_cached_stacks = 64;

// 未使用的coroutine stacks构成的stack
//
// 该stack用于快速分配coroutine stack，当一个coroutine被释放其之前的stack被放置在栈顶，
// 假如此时有新的coroutine创建，那么该stack对应的虚拟内存页面有极大的概率还在RAM中，
// 因此可以减少page miss的几率，快速分配coroutine stack的目的就达到了
static int mill_num_cached_stacks = 0;
static struct mill_slist mill_cached_stacks = {0};

// 分配coroutine stack，返回地址为stack+mill_stack_size，即栈顶（栈从高地址向低地址方向增长）
static void *mill_allocstackmem(void) {
    void *ptr;
#if defined HAVE_POSIX_MEMALIGN && HAVE_MPROTECT
    /* Allocate the stack so that it's memory-page-aligned. */
    int rc = posix_memalign(&ptr, mill_page_size(), mill_get_stack_size());
    if(mill_slow(rc != 0)) {
        errno = rc;
        return NULL;
    }
    /* The bottom page is used as a stack guard. This way stack overflow will
       cause segfault rather than randomly overwrite the heap. */
    rc = mprotect(ptr, mill_page_size(), PROT_NONE);
    if(mill_slow(rc != 0)) {
        int err = errno;
        free(ptr);
        errno = err;
        return NULL;
    }
#else
    ptr = malloc(mill_get_stack_size());
    if(mill_slow(!ptr)) {
        errno = ENOMEM;
        return NULL;
    }
#endif
    return (void*)(((char*)ptr) + mill_get_stack_size());
}

// 预分配coroutine stacks（分配count个栈尺寸为stack_size的协程栈）
void mill_preparestacks(int count, size_t stack_size) {
    // 释放cached的所有coroutine stack
    while(1) {
        struct mill_slist_item *item = mill_slist_pop(&mill_cached_stacks);
        if(!item)
            break;
        free(((char*)(item + 1)) - mill_get_stack_size());
    }
    // 现在没有分配的coroutine stacks，可以调整一下stack尺寸了
    size_t old_stack_size = mill_stack_size;
    size_t old_sanitised_stack_size = mill_sanitised_stack_size;
    mill_stack_size = stack_size;
    mill_sanitised_stack_size = 0;
    // 分配新的coroutine stacks并cache起来备用
    int i;
    for(i = 0; i != count; ++i) {
        void *ptr = mill_allocstackmem();
        if(!ptr)
            goto error;
        struct mill_slist_item *item = ((struct mill_slist_item*)ptr) - 1;
        mill_slist_push_back(&mill_cached_stacks, item);
    }
    mill_num_cached_stacks = count;
    // 确保这里分配的coroutine stacks不会被销毁，即便当前没有使用
    mill_max_cached_stacks = count;
    errno = 0;
    return;
error:
    // 如果无法分配所有的coroutine stacks，那就一个也不分配（已分配的释放），还原状态并返回错误
    while(1) {
        struct mill_slist_item *item = mill_slist_pop(&mill_cached_stacks);
        if(!item)
            break;
        free(((char*)(item + 1)) - mill_get_stack_size());
    }
    mill_num_cached_stacks = 0;
    mill_stack_size = old_stack_size;
    mill_sanitised_stack_size = old_sanitised_stack_size;
    errno = ENOMEM;
}

// 分配一个coroutine stack（先从cached stacks里面取，如果没有获取到再从内存分配）
void *mill_allocstack(size_t *stack_size) {
    if(!mill_slist_empty(&mill_cached_stacks)) {
        --mill_num_cached_stacks;
        return (void*)(mill_slist_pop(&mill_cached_stacks) + 1);
    }
    void *ptr = mill_allocstackmem();
    if(!ptr)
        mill_panic("not enough memory to allocate coroutine stack");
    if(stack_size)
        *stack_size = mill_get_stack_size();
    return ptr;
}

// 释放coroutine stack（参数stack为栈底）
// 如果当前cached stacks小于阈值则将当前待释放的stack cache起来，反之释放其内存
void mill_freestack(void *stack) {
    /* Put the stack to the list of cached stacks. */
    struct mill_slist_item *item = ((struct mill_slist_item*)stack) - 1;
    mill_slist_push_back(&mill_cached_stacks, item);
    if(mill_num_cached_stacks < mill_max_cached_stacks) {
        ++mill_num_cached_stacks;
        return;
    }
    /* We can't deallocate the stack we are running on at the moment.
       Standard C free() is not required to work when it deallocates its
       own stack from underneath itself. Instead, we'll deallocate one of
       the unused cached stacks. */
    item = mill_slist_pop(&mill_cached_stacks);
    void *ptr = ((char*)(item + 1)) - mill_get_stack_size();
#if HAVE_POSIX_MEMALIGN && HAVE_MPROTECT
    int rc = mprotect(ptr, mill_page_size(), PROT_READ|PROT_WRITE);
    mill_assert(rc == 0);
#endif
    free(ptr);
}
```

