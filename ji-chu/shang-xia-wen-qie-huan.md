# 上下文切换

## 上下文，指的是什么呢？

一个任务的执行，本质上就是CPU执行其指令的过程，每执行一条指令，CPU内部寄存器状态都会发生一点改变，CPU内部寄存器的状态，其实就标识了任务当前的一个执行状态。这里记录任务状态的CPU寄存器信息就是我们常说的上下文。

## 上下文，如何表示呢？

现在，我们了解了上下文其实就是记录任务执行状态的寄存器信息。那表示一个上下文，需要用到CPU中的哪些寄存器呢？全部吗？还是一部分？

CPU中的寄存器数量有多少呢，这个肯定跟具体型号的CPU有关系。参考Intel手册，[https://software.intel.com/content/www/us/en/develop/articles/intel-sdm.html](https://software.intel.com/content/www/us/en/develop/articles/intel-sdm.html)。寄存器种类大致上包括：通用目的寄存器（AX\BX\CX\DX...\)，段寄存器\(CS\DS\SS...\)，标识寄存器\(EFLAGS\)，指令\(IR\)及指令指针寄存器\(IP\)，当然还有一些其他的，比如SP\GDTR\LDTR\CR3等等。

对于表征任务当前的执行状态而言，其实是不需要使用到所有的寄存器的，因为有些寄存器的值是从进程内存地址空间中加载出来的，或者从内核数据结构中加载出来的，我们只需要那些表征任务当前执行状态的寄存器就可以了，比如常见的CS:IP，它记录了任务要执行的下一条指令的地址，CR3是进程页表起始地址，由内核从特定数据结构中读取并加载到CR3寄存器，这个就不需要作为上下文中的信息。

## 上下文，jmp\_buf

我们看下Linux中如何标识进程上下文信息。熟悉Linux下c语言开发的同学，应该知道setjmp, longjmp这两个函数。前者定义一个`jmp_buf`结构体，后者可以在其他函数中实现跨函数的跳转（goto只能实现函数间跳转）。这里的jmp\_buf其实就是表示的任务的栈上下文信息（stack context）。

以下是i386架构中 `struct __jmp_buf` 的定义，x86\_64架构就先不赘述了。

```c
// 处理器架构：i386 
// Linux/arch/x86/um/shared/sysdep/archsetjmp_32.h
struct __jmp_buf {
unsigned int __ebx; // 通用数据寄存器之一
unsigned int __esp; // 栈指针寄存器(进程栈空间由高地址向低地址方向增长)
unsigned int __ebp; // 基址指针寄存器(记录了当前栈帧的起始地址(进入一个函数后首先执行的便是push %ebp; mov %esp, %ebp))
unsigned int __esi; // 源变址寄存器
unsigned int __edi; // 目的变址寄存器
unsigned int __eip; // 指令指针寄存器(程序计数器PC=CS:IP,二者结合起来确定下一条待执行的机器指令地址)
};
typedef struct __jmp_buf jmp_buf[1];
```

如果之前有写过汇编或者反汇编的经历，对这些寄存器应该不会太陌生。函数调用开头一般就是 "push epb", "mov ebp, esp", "sub $size, esp"的操作，函数返回前也常看到ret这种对eip进行修改的指令。

简单说，其实就是这些寄存器基本描述了一个任务的当前执行状态，当这个任务被切换出去的时候，需要保存这些寄存器信息到进程PCB（task\_struct）中，等任务下次重新被调度的时候，再从进程PCB中读取并还原到寄存器，任务就可以不间断地继续执行下去。

现在，我们介绍了任务的上下文如何表示，也了解了上下文如何保存、恢复，我们还需要了解更多信息。

## 上下文，sigjmp\_buf

前面提过了jmp\_buf，jmp\_buf有个不足之处，就是它没有考虑信号处理掩码的问题，glibc在此基础上做了改进，定义了一个新的类型`sigjmp_buf`，对应的也提供了函数sigsetjmp/siglongjmp。

```c
struct __jmp_buf_tag
{
/* NOTE: The machine-dependent definitions of `__sigsetjmp'
assume that a `jmp_buf' begins with a `__jmp_buf' and that
`__mask_was_saved' follows it. Do not move these members
or add others before it. */
__jmp_buf __jmpbuf; /* Calling environment. */
int __mask_was_saved; /* Saved the signal mask? */
__sigset_t __saved_mask; /* Saved signal mask. */
};
typedef struct __jmp_buf_tag jmp_buf[1];
```

sigjmp\_buf不仅保存了处理器硬件上下文信息（jmp\_buf），还保存了信号掩码，这些共同构成了任务的上下文信息。

## 上下文，ucontext\_t

在经历了jmp\_buf、sigjmp\_buf这样的改进之后，还是不能完全满足要求，我们还需要更加精细有力的上下文切换控制，ucontext\_t诞生了。

在System-V系统中，ucontext.h提供了两个新类型和4个新函数。

上下文对应的类型mcontext\_t是与机器相关的、不透明的，它作为ucontext\_t的一个内部成员。ucontext\_t定义则至少包含了如下字段：

```c
typedef struct ucontext {
    // pointer to the context that will be resumed when this context returns
    struct ucontext *uc_link;
    
    // the set of signals that are blocked when this context is active
    sigset_t         uc_sigmask;
    
    // the stack used by this context
    stack_t          uc_stack;
    
    // a machine-specific representation of the saved context
    mcontext_t       uc_mcontext;
    ...
} ucontext_t;
```

上下文相关的4个操作函数：

```c
int  getcontext(ucontext_t *);
int  setcontext(const ucontext_t *);
void makecontext(ucontext_t *, (void *)(), int, ...);
int  swapcontext(ucontext_t *, const ucontext_t *);
```

wikipedia提供了一个非常赞的示例，来演示如何实现任务切换，如下所示。没有接触过这些的可能开始有点懵，我来解释下这里的逻辑。

程序中创建了3个ucontext\_t变量，main\_context1在68行被设置，loop函数退出时loop函数绑定的loop\_context将被销毁，程序中会隐式调用setcontext\(loop\_context.uc\_link\)来切换到父上下文中去执行，即getcontext\(&main\_context1\)对应的下一条指令地址处（70行）。

main函数执行时，什么时候激活loop函数呢，在while循环内swapcontext的时候（80行），执行到这里的时候，main函数将当前上下文信息存储到main\_context2，并切换到loop\_context去执行，执行什么呢？50行~61行定义了loop\_context的父上下文，以及loop\_context绑定的函数执行时要采用的栈空间，当然还绑定了要执行的函数loop以及传递给loop的参数信息。loop函数将使用数组iterator\_stack\[SIGSTKSZ\]作为自己的栈空间。

loop执行起来之后，会进入一个循环，每轮循环会更新一下变量i\_from\_iterator的值，然后呢，就执行上下文切换swapcontext\(loop\_context, other\_context\)，这里的other\_context刚好是main\_context2，切过去之后就是执行语句printf\("%d\n", i\_from\_iterator\)来完成变量的打印。然后进入while\(1\)循环，此时又会执行上下文切换继续执行loop函数，loop函数从上次停下来的地方继续执行，这样连续打印了0~9。

然后，loop函数for循环结束，退出函数，loop\_context结束，此时函数退出前会隐式调用setcontext以切换到父上下文main\_context1去执行，由于前面main函数执行时已经将变量iterator\_finished更新为了1，所以此时即便从main\_context1继续执行，也不会再次进入while循环再次执行loop函数。

程序正常结束。

```c
#include <stdio.h>
#include <stdlib.h>
#include <ucontext.h>

/* The three contexts:
 *    (1) main_context1 : The point in main to which loop will return.
 *    (2) main_context2 : The point in main to which control from loop will
 *                        flow by switching contexts.
 *    (3) loop_context  : The point in loop to which control from main will
 *                        flow by switching contexts. */
ucontext_t main_context1, main_context2, loop_context;

/* The iterator return value. */
volatile int i_from_iterator;

/* This is the iterator function. It is entered on the first call to
 * swapcontext, and loops from 0 to 9. Each value is saved in i_from_iterator,
 * and then swapcontext used to return to the main loop.  The main loop prints
 * the value and calls swapcontext to swap back into the function. When the end
 * of the loop is reached, the function exits, and execution switches to the
 * context pointed to by main_context1. */
void loop(
    ucontext_t *loop_context,
    ucontext_t *other_context,
    int *i_from_iterator)
{
    int i;
    
    for (i=0; i < 10; ++i) {
        /* Write the loop counter into the iterator return location. */
        *i_from_iterator = i;
        
        /* Save the loop context (this point in the code) into ''loop_context'',
         * and switch to other_context. */
        swapcontext(loop_context, other_context);
    }
    
    /* The function falls through to the calling context with an implicit
     * ''setcontext(&loop_context->uc_link);'' */
} 
 
int main(void)
{
    /* The stack for the iterator function. */
    char iterator_stack[SIGSTKSZ];

    /* Flag indicating that the iterator has completed. */
    volatile int iterator_finished;

    getcontext(&loop_context);
    /* Initialise the iterator context. uc_link points to main_context1, the
     * point to return to when the iterator finishes. */
    loop_context.uc_link          = &main_context1;
    loop_context.uc_stack.ss_sp   = iterator_stack;
    loop_context.uc_stack.ss_size = sizeof(iterator_stack);

    /* Fill in loop_context so that it makes swapcontext start loop. The
     * (void (*)(void)) typecast is to avoid a compiler warning but it is
     * not relevant to the behaviour of the function. */
    makecontext(&loop_context, (void (*)(void)) loop,
        3, &loop_context, &main_context2, &i_from_iterator);
   
    /* Clear the finished flag. */      
    iterator_finished = 0;

    /* Save the current context into main_context1. When loop is finished,
     * control flow will return to this point. */
    getcontext(&main_context1);
  
    if (!iterator_finished) {
        /* Set iterator_finished so that when the previous getcontext is
         * returned to via uc_link, the above if condition is false and the
         * iterator is not restarted. */
        iterator_finished = 1;
       
        while (1) {
            /* Save this point into main_context2 and switch into the iterator.
             * The first call will begin loop.  Subsequent calls will switch to
             * the swapcontext in loop. */
            swapcontext(&main_context2, &loop_context);
            printf("%d\n", i_from_iterator);
        }
    }
    
    return 0;
}
```

我们可以通过 `gcc -o main main.c` 编译并且通过 `./main` 来运行，运行会得到如下结果：

```text
$ ./main 
0
1
2
3
4
5
6
7
8
9
```

这里的volatile qualifier是为了避免编译器相关的优化，在这个示例中作用不大，如果是多线程+强一致CPU中还有点用。关于volatile的细节可以参考我的另一篇文章：[https://hitzhangjie.github.io/blog/2019-01-07-%E4%BD%A0%E4%B8%8D%E8%AE%A4%E8%AF%86%E7%9A%84cc++-volatile/](https://hitzhangjie.github.io/blog/2019-01-07-%E4%BD%A0%E4%B8%8D%E8%AE%A4%E8%AF%86%E7%9A%84cc++-volatile/)。

总结一下，ucontext\_t提供了更灵活的上下文切换控制，是比jmp\_buf/sigjmp\_buf更好的选择。ucontext\_t的设计初衷，也是为了支持实现一些更加高级的流程控制、fiber、coroutine等。所以协程并不是什么新概念，它只是一种调度实体粒度大小的划分方式。

如果只实现协程其实也没什么大用，还需要大量的其他基础设施，如协程之间如何通信，如何做到同步、互斥，等等，这些才是决定好用不好用的问题。所以我们也能理解为什么协程库有五花八门的实现，但是没有哪个很出名的，但是go却能独领风骚。大致就是这个原因吧。

言归正传，我们是否可以直接利用ucontext\_t来实现协程呢？我们还需要考虑下上下文切换的开销才能决定。ucontext\_t的目的是通用，因此其内部mcontext\_t包含了所有的CPU寄存器信息，有些是没必要的，如果每次上下文切换都把寄存器信息保存一下、恢复一下，这里的开销我们人类感觉不到，但是机器是能感觉到的，必须要考虑下，这部分内容我们在下一节介绍。

当前比较火的libgo协程库，之前也是基于ucontext\_t来实现的，不过后面的版本做了些优化，去掉了一些冗余的寄存器的保存、恢复操作，可以参考libgo patches setcontext/getcontext。

> Google's Ian Lance Taylor explained in the commit improving libgo, "Currently, goroutine switches are implemented with libc getcontext/setcontext functions, which saves/restores the machine register states and also the signal context. This does more than what we need, and performs an expensive syscall. This CL implements a simplified version of getcontext/setcontext, in assembly, that only saves/restores the necessary part, i.e. the callee-save registers, and the PC, SP. A simplified version of makecontext, written in C, is also added. Currently this is only implemented on Linux/AMD64."



参考资料：

1. setcontext, [https://en.wikipedia.org/wiki/Setcontext](https://en.wikipedia.org/wiki/Setcontext)
2. libgo cheaper context switch on x86\_64,  [https://gcc.gnu.org/legacy-ml/gcc-patches/2019-05/msg02155.html](https://gcc.gnu.org/legacy-ml/gcc-patches/2019-05/msg02155.html)
3. linux man\(2\)













