---
description: >-
  libmill是一个基于c语言开发的go风格协程库，实现了go风格的协程操作、chan操作、非阻塞的网络io操作等等，是一个不错的linux平台下c风格协程库实现。如果你想从0到1的快速了解如何开发一个协程库，libmill将是一个非常不错的案例；如果你想更深入地了解go，libmill里面也借鉴了go的一些设计思想；或者你想在生产环境中使用，开发者也提供了一个更健壮的版本libdill。
---

# 常用工具函数：函数+宏

#### utils

```c
// ptr是指向结构体type中成员member的指针，计算包含该member的结构体的地址
// - 在list等实现中，mill_cont用于获取“迭代器”对应的元素结构体地址
#define mill_cont(ptr, type, member) \
    (ptr ? ((type*) (((char*) ptr) - offsetof(type, member))) : NULL)
```

```c
// 编译时断言
#define MILL_CT_ASSERT_HELPER2(prefix, line)    prefix##line
#define MILL_CT_ASSERT_HELPER1(prefix, line)    MILL_CT_ASSERT_HELPER2(prefix, line)
#define MILL_CT_ASSERT(x) \
    typedef int MILL_CT_ASSERT_HELPER1(ct_assert_,__COUNTER__) [(x) ? 1 : -1]
```

```c
// 分支判断，便于编译器分支预测
#if defined __GNUC__ || defined __llvm__
#define mill_fast(x) __builtin_expect(!!(x), 1)
#define mill_slow(x) __builtin_expect(!!(x), 0)
#else
#define mill_fast(x) (x)
#define mill_slow(x) (x)
#endif
```

```c
// 自定义断言
#define mill_assert(x) \
    do {\
        if (mill_slow(!(x))) {\
            fprintf(stderr, "Assert failed: " #x " (%s:%d)\n",\
                __FILE__, __LINE__);\
            fflush(stderr);\
            abort();\
        }\
    } while (0)
#endif
```

