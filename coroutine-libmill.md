---
description: >-
  libmill是一个基于c语言开发的go风格协程库，实现了go风格的协程操作、chan操作、非阻塞的网络io操作等等，是一个不错的linux平台下c风格协程库实现。如果你想从0到1的快速了解如何开发一个协程库，libmill将是一个非常不错的案例；如果你想更深入地了解go，libmill里面也借鉴了go的一些设计思想；或者你想在生产环境中使用，开发者也提供了一个更健壮的版本libdill。
---

# coroutine: libmill.h

**file: libmill.h**

**0 ABI版本**

```c
// ABI应用程序二进制接口
#define MILL_VERSION_CURRENT 19         // 主版本
#define MILL_VERSION_REVISION 1         // 修订版本
#define MILL_VERSION_AGE 1              // 支持过去的几个版本
```

**1 符号可见性**

libmill这里是要编译成共享库的，共享库应该将实现相关的细节屏蔽，只暴露接口给外部调用就好，因此在共享库中有个“**符号可见性**”的问题（可参考下面的引文）。该工程在在Makefile里通过编译器选项`-fvisibility inlines=hidden`来设置默认对外隐藏工程中的符号，对于提供给外部使用的接口使用可见性属性`__attribute__((visibility("default")))`来单独设置其可见性。

> Functions with default visibility have a global scope and can be called from other shared objects. Functions with hidden visibility have a local scope and cannot be called from other shared objects. Visibility can be controlled by using either compiler options or visibility attributes. 更多关于符号可见性的描述，可以参考：[点击查看](https://www.ibm.com/support/knowledgecenter/en/SSB23S_1.1.0.14/gtpl2/export.html)。

```c
#if !defined __GNUC__ && !defined __clang__
#error "Unsupported compiler!"
#endif

#if defined MILL_NO_EXPORTS
#   define MILL_EXPORT
#else
#   if defined _WIN32
#      ......
#   else
#      if defined __SUNPRO_C
#          ......
#      elif (defined __GNUC__ && __GNUC__ >= 4) || defined __INTEL_COMPILER || defined __clang__
#          define MILL_EXPORT __attribute__ ((visibility("default")))
#      else
#          define MILL_EXPORT
#      endif
#   endif
#endif
```

如果函数名前有宏**MILL\_EXPORT**，表示该函数具有默认可见性，可在libmill.so外的代码中被调用。这里我们举个两字来说明一下：

```c
// mill_concat_可见性为Makefile中指定的hidden，只能在当前so中使用，对外不可见
#define mill_concat_(x,y) x##y

// mill_now_、mill_mfork_可见性为default，对外可见，可在so外部调用
MILL_EXPORT int64_t mill_now_(void);
MILL_EXPORT pid_t mill_mfork_(void);
```

libmill.h中涉及到大量的函数名导出的问题，这里由于篇幅的原因不再一一列出。

**2 定时器精度**

获取系统时间函数**gettimeofday**还是比较耗时的，对于频繁需要获取系统时间的情景下，需要对获取到的系统时间做一定的cache。为了保证时间精度，这里的cache更新时间必须要控制好。

如何决定何时更新cache的系统时间呢？**rdtsc**（read timestamp counter）指令执行只需要几个时钟周期，它返回系统启动后经过的时钟周期数。这里可以根据CPU频率指定一个时钟周期数量作为阈值，当前后两次rdtsc读取到的时钟周期数的差值超过这个阈值再调用gettimeofday来更新系统时间。这里libmill中的定时器timer就是这么实现的。

下面是**rdtsc**指令的简要说明，详情请查看：[wiki rdtsc](https://en.wikipedia.org/wiki/Time_Stamp_Counter)。

> The Time Stamp Counter \(TSC\) is a 64-bit register present on all x86 processors since the Pentium. It counts the number of cycles since reset. The instruction RDTSC returns the TSC in EDX:EAX. In x86-64 mode, RDTSC also clears the higher 32 bits of RAX and RDX. Its opcode is 0F 31.\[1\] Pentium competitors such as the Cyrix 6x86 did not always have a TSC and may consider RDTSC an illegal instruction. Cyrix included a Time Stamp Counter in their MII.

该头文件涉及源码较多，这里只列出了比较关键的代码，感兴趣的可以查看源代码来了解更多细节。

**3 mill\_fdwait关注的io事件**

```c
// mill_fdwait关注的读写错误事件
#define MILL_FDW_IN_ 1
#define MILL_FDW_OUT_ 2
#define MILL_FDW_ERR_ 4
```

**4 协程上下文的保存和切换**

**4.0 协程上下文**

```c
#if defined __x86_64__
typedef uint64_t *mill_ctx;
#else
typedef sigjmp_buf *mill_ctx;
#endif
```

**4.1 协程上下文的保存**

```c
// x86_64平台下协程上下文保存实现
#if defined(__x86_64__)
......
// 保存当前协程运行时上下文（就是保存处理器硬件上下文到指定内存区域ctx中备用）
//
// 这里使用宏来实现可以避免函数调用堆栈创建、销毁带来的开销，实现更高效地协程切换，
// linux gcc内联汇编，汇编相关参数可以分为“指令部”、“输出部”、“输入部”、“破坏部”，
// 这里将内存变量ctx的值传入寄存器rdx，并将最后rax寄存器的值赋值给变量ret，
// 指令将rax清零，将rbx、r12、rsp、r13、r14、r15、rcx、rdi、rsi依次保存到ctx为起始地址的内存中
#define mill_setjmp_(ctx) ({\
    int ret;\
    asm("lea     LJMPRET%=(%%rip), %%rcx\n\t"\  //==>返回地址(LJMPRET标号处)送入%%rcx
        "xor     %%rax, %%rax\n\t"\
        "mov     %%rbx, (%%rdx)\n\t"\
        "mov     %%rbp, 8(%%rdx)\n\t"\
        "mov     %%r12, 16(%%rdx)\n\t"\
        "mov     %%rsp, 24(%%rdx)\n\t"\
        "mov     %%r13, 32(%%rdx)\n\t"\
        "mov     %%r14, 40(%%rdx)\n\t"\
        "mov     %%r15, 48(%%rdx)\n\t"\
        "mov     %%rcx, 56(%%rdx)\n\t"\         //==>rcx又存储到56(%%rdx)
        "mov     %%rdi, 64(%%rdx)\n\t"\
        "mov     %%rsi, 72(%%rdx)\n\t"\
        "LJMPRET%=:\n\t"\
        : "=a" (ret)\
        : "d" (ctx)\
        : "memory", "rcx", "r8", "r9", "r10", "r11",\
          "xmm0", "xmm1", "xmm2", "xmm3", "xmm4", "xmm5", "xmm6", "xmm7",\
          "xmm8", "xmm9", "xmm10", "xmm11", "xmm12", "xmm13", "xmm14", "xmm15"\
          MILL_CLOBBER\
          );\
    ret;\
})
```

**4.2 协程上下文的恢复**

```c
// x86_64平台下协程上下文恢复实现

#if defined(__x86_64__)
......
// 恢复协程上下文信息到处理器中（从ctx开始的内存区域中加载之前保存的处理器硬件上下文）
//
// 要恢复某个协程cr的运行时，先获取其挂起之前保存的上下文cr->ctx，然后mill_longjmp(ctx)即可，
// 将ctx值加载到rax，采用相对寻址依次加载ctx为起始地址的内存区域中保存的上下文信息到寄存器，
// 最后回复执行
#define mill_longjmp_(ctx) \
    asm("movq   (%%rax), %%rbx\n\t"\
        "movq   8(%%rax), %%rbp\n\t"\
        "movq   16(%%rax), %%r12\n\t"\
        "movq   24(%%rax), %%rdx\n\t"\
        "movq   32(%%rax), %%r13\n\t"\
        "movq   40(%%rax), %%r14\n\t"\
        "mov    %%rdx, %%rsp\n\t"\
        "movq   48(%%rax), %%r15\n\t"\
        "movq   56(%%rax), %%rdx\n\t"\     //==>56(%%rax)中地址为返回地址，送入%%rdx
        "movq   64(%%rax), %%rdi\n\t"\
        "movq   72(%%rax), %%rsi\n\t"\
        "jmp    *%%rdx\n\t"\
        : : "a" (ctx) : "rdx" \
    )
#else
// 非x86_64要借助sigsetjmp\siglongjmp来实现协程上下文切换
#define mill_setjmp_(ctx) \
    sigsetjmp(*ctx, 0)
#define mill_longjmp_(ctx) \
    siglongjmp(*ctx, 1)
#endif
```

**5 go\(func\)实现**

go\(func\)的作用是挂起当前协程，并在一个新创建的协程中运行指定的函数func，func执行完成后再销毁新创建的协程，并重新调度其他协程运行（也包括当前协程）。

mill_go_\(fn\)的实现，我花了不少时间才看懂，这里还要感谢libmill贡献者[**raedwulf**](https://github.com/raedwulf)对我的帮助，通过交流他一下就明白了我困惑的源头并给我指出了不该忽略的关键4行代码！

```c
...

int mill_anchor[mill_unoptimisable1_];\
mill_unoptimisable2_ = &mill_anchor;\
char mill_filler[(char*)&mill_anchor - (char*)(mill_sp)];\
mill_unoptimisable2_ = &mill_filler;\

...
```

这4行代码确实令人困惑，Stack Overflow上的朋友们看到提问的这个问题甚至给踩了好几次，同事看后也觉得有点无厘头，也难怪被raedwulf戏称为black magic around c language！

```c
// go()的实现
#define mill_go_(fn) \
    do {\
        void *mill_sp;\
        // 获取当前正在运行的协程上下文，并及时进行保存，因为我们马上要调整栈帧了
        mill_ctx ctx = mill_getctx_();\
        if(!mill_setjmp_(ctx)) {\
            // 为即将新创建的协程分配对应的内存空间，并返回stack部分的当前栈顶（等于栈底）位置
            mill_sp = mill_prologue_(MILL_HERE_);\
            // 下面4行代码困扰了我很久，原因还是对linux c、gcc、栈帧分配不够精通。
            // - 栈帧中的空间分配，对于编译时可确定尺寸的就编译时分配，通过sub <size>,%rsp来实现；
            // - 栈帧中的空间分配，运行时才可以确定的就需要运行时分配，通过alloca(size)来实现；
            // 注意：
            // - gcc在x86_64下alloca的工作主要是sub <size>,%rsp外加一些内存对齐的操作，alloca在当
            //   前栈帧中分配空间并返回空间起始地址，但是不检查栈是否越界；
            // - 另外指针运算是无符号计算，小地址减去大地址的结果会在整个虚拟内存地址空间中滚动；
            // - mill_filler数组的分配是由alloca完成，分配完成后rsp将被调整为mill_sp指向的内存空间；
            // - 新的协程将以mill_sp作为当前栈顶运行，等当前协程恢复上下文并运行时，其根本意识不
            //   到该所谓的mill_filler的存在，因为保存其上下文操作是早于栈调整操作的；
            int mill_anchor[mill_unoptimisable1_];\
            mill_unoptimisable2_ = &mill_anchor;\
            char mill_filler[(char*)&mill_anchor - (char*)(mill_sp)];\
            mill_unoptimisable2_ = &mill_filler;\
            // 在新创建的协程栈空间中调用函数fn
            fn;\
            // fn执行结束后释放占用的协程内存空间，并mill_suspend让出cpu给其他协程
            mill_epilogue_();\
        }\
    } while(0)
```

**6 chan实现**

**6.0 常用数据结构**

```c
// 这里只是声明，定义在cr.h中
struct mill_chan_;

typedef struct{
    void *f1; 
    void *f2; 
    void *f3; 
    void *f4; 
    void *f5; 
    void *f6; 
    int f7; 
    int f8; 
    int f9;
} mill_clause_;

#define MILL_CLAUSELEN_ (sizeof(mill_clause_))
```

**6.1 发送数据到chan**

```c
// 发送type类型的数据value到channel
#define mill_chs__(channel, type, value) \
    do {\
        type mill_val = (value);\
        mill_chs_((channel), &mill_val, sizeof(type), MILL_HERE_);\
    } while(0)
```

**6.2 从chan接收数据**

```c
// 从channel接收type类型的数据
#define mill_chr__(channel, type) \
    (*(type*)mill_chr_((channel), sizeof(type), MILL_HERE_))
```

**6.3 chan操作结束**

```c
// 用type类型数据value来标记channel操作结束
#define mill_chdone__(channel, type, value) \
    do {\
        type mill_val = (value);\
        mill_chdone_((channel), &mill_val, sizeof(type), MILL_HERE_);\
    } while(0)
```

**7 choose从句实现**

**7.1 从句初始化**

```c
#define mill_choose_init__ \
    {\
        mill_choose_init_(MILL_HERE_);\
        int mill_idx = -2;\
        while(1) {\
            if(mill_idx != -2) {\
                if(0)
```

**7.2 读就绪事件**

```c
#define mill_choose_in__(chan, type, name, idx) \
                    break;\
                }\
                goto mill_concat_(mill_label, idx);\
            }\
            char mill_concat_(mill_clause, idx)[MILL_CLAUSELEN_];\
            mill_choose_in_(\
                &mill_concat_(mill_clause, idx)[0],\
                (chan),\
                sizeof(type),\
                idx);\
            if(0) {\
                type name;\
                mill_concat_(mill_label, idx):\
                if(mill_idx == idx) {\
                    name = *(type*)mill_choose_val_(sizeof(type));\
                    goto mill_concat_(mill_dummylabel, idx);\
                    mill_concat_(mill_dummylabel, idx)
```

**7.3 写就绪事件**

```c
#define mill_choose_out__(chan, type, val, idx) \
                    break;\
                }\
                goto mill_concat_(mill_label, idx);\
            }\
            char mill_concat_(mill_clause, idx)[MILL_CLAUSELEN_];\
            type mill_concat_(mill_val, idx) = (val);\
            mill_choose_out_(\
                &mill_concat_(mill_clause, idx)[0],\
                (chan),\
                &mill_concat_(mill_val, idx),\
                sizeof(type),\
                idx);\
            if(0) {\
                mill_concat_(mill_label, idx):\
                if(mill_idx == idx) {\
                    goto mill_concat_(mill_dummylabel, idx);\
                    mill_concat_(mill_dummylabel, idx)
```

**7.3 deadline实现**

```c
#define mill_choose_deadline__(ddline, idx) \
                    break;\
                }\
                goto mill_concat_(mill_label, idx);\
            }\
            mill_choose_deadline_(ddline);\
            if(0) {\
                mill_concat_(mill_label, idx):\
                if(mill_idx == -1) {\
                    goto mill_concat_(mill_dummylabel, idx);\
                    mill_concat_(mill_dummylabel, idx)
```

**7.4 otherwise实现**

```c
#define mill_choose_otherwise__(idx) \
                    break;\
                }\
                goto mill_concat_(mill_label, idx);\
            }\
            mill_choose_otherwise_();\
            if(0) {\
                mill_concat_(mill_label, idx):\
                if(mill_idx == -1) {\
                    goto mill_concat_(mill_dummylabel, idx);\
                    mill_concat_(mill_dummylabel, idx)
```

**7.5 从句结束**

```c
#define mill_choose_end__ \
                    break;\
                }\
            }\
            mill_idx = mill_choose_wait_();\
        }
```

