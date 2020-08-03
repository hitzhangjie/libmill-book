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

wikipedia提供了一个非常赞的示例：

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











