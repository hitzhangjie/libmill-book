---
description: >-
  理解进程地址空间布局、作用、设置方式，是我们深入理解多任务的基础。这里我们从进程地址空间入手，先介绍下“栈”的基础知识、栈的分配方式，然后我们再重点介绍与协程运行紧密相关的协程栈空间的分配方式。理解了协程栈空间的分配之后，我们将再下一节再进一步探究任务上下文相关的内容。
---

# 栈空间分配

## 栈基础知识

现在的计算机体系结构多数是基于栈的架构，ABI决定了函数调用的规范，调用函数时如何调用、如何传参、如何返回值等。每当发生一次函数调用就会创建一个对应的栈帧，栈帧相互之间是隔离的，当前栈帧中的内存读写操作不会影响到调用函数，因为当前函数return时栈帧也就销毁了。

栈帧相当于为函数计算提供了一个隔离的计算环境，简言之，栈帧是很有效的。以Linux系统为例，进程的虚拟地址空间布局组织如下，我们通常说的栈就是指的 “**User Stack**” 区域。一个进程的最大栈空间是有限制的，可通过 “**ulimit -s**”来查看、设置，对于线程（非主线程）其地址空间是位于堆、栈之间的mmap区域，线程栈大小也是有限制的，默认2MB。栈空间里面，就是函数调用时生成的一系列栈帧。

![Linux&#x8FDB;&#x7A0B;&#x865A;&#x62DF;&#x5185;&#x5B58;&#x5730;&#x5740;&#x7A7A;&#x95F4;&#x5E03;&#x5C40;](../.gitbook/assets/image%20%2810%29.png)

下文为了描述方便，统一用栈空间代指完整的栈空间或者部分栈帧的空间，读者可根据语境自行区分。

## 栈空间分配

栈空间的分配有两个时机，一个是编译时，一个是运行时：

* 编译时可以确定栈空间大小的就在编译时生成对应的汇编指令来分配，如：`sub 0x16, %rsp`；
* 运行时才可以确定栈空间大小的就要在运行时分配，如：`int n = srand()%16; int buf[n];。`

这里，我们主要关注下运行时栈空间分配，因为它和libmill协程栈的分配、组织有关系。先看这里的示例，`int n = srand()%16; int buf[n];` ，这里数组的栈空间如何分配呢？这里的变量`n`的值只有在程序运行时才可以确定，所以这里的 `int buf[n];` 肯定是在运行时动态分配栈空间的，我来解释下。

以如下源程序为例，代码很简单，运行时获取一个随机数n，用来作为动态创建数组的大小。

file: main.c

```c
#include <stdio.h>
#include <stdlib.h>

int main(int argc, char *argv[])
{
    int n = rand();
    int buf[n*100];
    buf[0] = 1;
    buf[n*100-1] = 0;

    return 0;
}

```

运行命令`gcc -g -O0 -o main main.c`，然后运行\`objdump -dS main\`得到如下反汇编后的指令，如果读者有汇编基础的话，很容易就看懂了。前面我们也有提及，分配栈帧时通常通过对寄存器rsp的调整来实现，因为栈是从高地址向低地址方向增长的，所以对栈的分配动作其实是sub指令对rsp寄存器做减法操作，重点关注虚线“---”标注的部分就可以了。

这里通过随机数n动态分配数组，最终也是通过subq对寄存器rsp做减法操作，只是在做减法操作之前，需要先确保有足够的栈空间，因此有条指令 `callq 96 <main.c+0x10000f6e>`，该指令其实是`chkstk_darwin`进行栈越界检查并在必要时触发pagefault向内核申请新的物理内存并建立映射。

我是如何得知callq这条指令调用的是chkstk\_darwin的呢？您可以通过 `gcc -S main.c` 来求证这点。

```text
Disassembly of section __TEXT,__text:

0000000100000ec0 _main:
; {
100000ec0: 55                           pushq   %rbp
100000ec1: 48 89 e5                     movq    %rsp, %rbp
100000ec4: 48 83 ec 40                  subq    $64, %rsp
100000ec8: 48 8b 05 31 01 00 00         movq    305(%rip), %rax
100000ecf: 48 8b 00                     movq    (%rax), %rax
100000ed2: 48 89 45 f8                  movq    %rax, -8(%rbp)
100000ed6: c7 45 f4 00 00 00 00         movl    $0, -12(%rbp)
100000edd: 89 7d f0                     movl    %edi, -16(%rbp)
100000ee0: 48 89 75 e8                  movq    %rsi, -24(%rbp)
-------------------------------------------------------------------------
;     int n = rand();
100000ee4: e8 91 00 00 00               callq   145 <main.c+0x100000f7a>
100000ee9: 89 45 e4                     movl    %eax, -28(%rbp)
-----------------------------------------\ 函数返回值存储到变量n中 \---------
;     int buf[n*100];
100000eec: 8b 45 e4                     movl    -28(%rbp), %eax
100000eef: 6b c0 64                     imull   $100, %eax, %eax
100000ef2: 89 c1                        movl    %eax, %ecx
100000ef4: 48 89 e2                     movq    %rsp, %rdx
100000ef7: 48 89 55 d8                  movq    %rdx, -40(%rbp)
100000efb: 48 89 ca                     movq    %rcx, %rdx
100000efe: 48 c1 e2 02                  shlq    $2, %rdx
100000f02: 48 89 d0                     movq    %rdx, %rax
100000f05: 48 89 4d c8                  movq    %rcx, -56(%rbp)
-------------------------------------------------------------------------
100000f09: e8 60 00 00 00               callq   96 <main.c+0x100000f6e>
-----------------------------------------\chkstk_darwin检查是否缺页异常\----
100000f0e: 48 29 c4                     subq    %rax, %rsp
-----------------------------------------\申请栈空间来创建数组(赋值时才会)\---
100000f11: 48 89 e0                     movq    %rsp, %rax
100000f14: 48 8b 4d c8                  movq    -56(%rbp), %rcx
100000f18: 48 89 4d d0                  movq    %rcx, -48(%rbp)
;     buf[0] = 1;
100000f1c: c7 00 01 00 00 00            movl    $1, (%rax)
;     buf[n*100-1] = 0;
100000f22: 6b 7d e4 64                  imull   $100, -28(%rbp), %edi
100000f26: 83 ef 01                     subl    $1, %edi
100000f29: 48 63 d7                     movslq  %edi, %rdx
100000f2c: c7 04 90 00 00 00 00         movl    $0, (%rax,%rdx,4)
;     return 0;
100000f33: c7 45 f4 00 00 00 00         movl    $0, -12(%rbp)
; }
```

## 协程栈空间

为了能创建很多的协程，协程栈空间就需要创建在内存范围更大的堆区。那如何在堆区创建协程的栈空间呢？其实很简单，rsp指向哪里，哪里就是当前的栈。

协程，只是一个普通的用户级可调度实体，协程还是最终运行在线程上的，执行过程中遇到函数、变量之类的需要栈空间辅助的，那还是要分配栈空间，这个和进程、线程完全相同，唯一的区别就是我们代替操作系统指明了可供分配栈空间的位置。不是进程栈区，而是在堆区的某一块申请到的内存中。我们所要做的就是将rsp指向这块堆区中的内存而已。

Linux下有个库函数**alloca**可以在当前栈帧上继续分配空间，但是呢，它不会检查是否出现越界的行为，注意了，因为内存分配后，栈顶会发生变化，寄存器%rsp会受到影响，基于这个side effect，我们就可以让rsp指向我们预先分配的堆区内存。

```bash
       alloca - allocate memory that is automatically freed

SYNOPSIS
       #include <alloca.h>

       void *alloca(size_t size);

DESCRIPTION
       The  alloca() function allocates size bytes of space in the stack frame 
       of the caller.  This temporary space is automatically freed when the 
       function that called alloca() returns to its caller.

RETURN VALUE
       The alloca() function returns a pointer to the beginning of the allocated 
       space.  If the allocation causes stack overflow, program behavior is 
       undefined.

CONFORMING TO
       This function is not in POSIX.1-2001.

       There is evidence that the alloca() function appeared in 32V, PWB, PWB.2, 
       3BSD, and 4BSD. There is a man page for it in 4.3BSD.  
       Linux uses the GNU version.

NOTES
       The alloca() function is machine- and compiler-dependent. For certain 
       applications, its use can improve efficiency compared to the use of malloc(3) 
       plus free(3). In certain  cases, it can also simplify memory deallocation 
       in applications that use longjmp(3) or siglongjmp(3). Otherwise, its use 
       is discouraged.

       Because the space allocated by alloca() is allocated within the stack frame, 
       that space is automatically freed if the function return is jumped over by 
       a call to longjmp(3) or siglongjmp(3).

       Do not attempt to free(3) space allocated by alloca()!
       ...

BUGS
       There is no error indication if the stack frame cannot be extended.  
       (However, after a failed allocation, the program is likely to receive 
       a SIGSEGV signal if it attempts to access the  unallocated space.)

       On  many systems alloca() cannot be used inside the list of arguments 
       of a function call, because the stack space reserved by alloca() would 
       appear on the stack in the middle of the space forthe function arguments.

SEE ALSO
       brk(2), longjmp(3), malloc(3)
```

libmill里面也是基于此原理，实现让指定的函数go\(func\)将新分配的内存空间当做自己的栈帧继续运行。这样每个协程都有自己的栈空间，再存储一下协程上下文就可以很方便地实现协程切换。如果你看不懂下面的代码，请联系下上面提及的示例。c语言黑魔法，哈哈哈！

```c
#define mill_go_(fn) \
    do {\
        void *mill_sp;\
        mill_ctx ctx = mill_getctx_();\
        if(!mill_setjmp_(ctx)) {\
            mill_sp = mill_prologue_(MILL_HERE_);\
            int mill_anchor[mill_unoptimisable1_];\
            mill_unoptimisable2_ = &mill_anchor;\
            char mill_filler[(char*)&mill_anchor - (char*)(mill_sp)];\
            mill_unoptimisable2_ = &mill_filler;\
            fn;\
            mill_epilogue_();\
        }\
    } while(0)
```

接下来我们会介绍下上下文切换相关的内容，为协程切换相关内容做准备。



