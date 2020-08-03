# 上下文切换开销

## 上下文切换有哪些开销

上下文切换，主要就是任务切换时任务状态的保存，以及选择下一个待执行的任务并将其上下文恢复。这其中的开销，主要包含这么几部分：

### 切换直接引入的开销

这部分是比较好理解的，保存、恢复上下文时需要保存的寄存器的数据量，直接决定了这部分开销。有些寄存器对上下文切换开销影响是比较大的，比如FPU、MMX、SSE、AVX等寄存器的状态，由于其数据量是比较大的，可能会增加及KB的数据，特别是AVX寄存器，AVX2有512字节，AVX-512有2KB，这么大的数据量在上下文切换时都需要读写内存，这里的读写延迟引入的开销是比较大的。

现在也存在“延迟状态加载（lazy state load）”这样的机制，当FPU、MMX、SSE、AVX寄存器没有加载数据时，能够避免加载其中的部分或者全部寄存器数据，从而一定程度上降低开销。如果几乎所有的任务都使用了这些寄存器状态信息，那么延迟状态加载，这个逻辑反而会变成累赘，增大开销。为啥呢？因为上下文切换过程中“按需加载寄存器的判断、执行逻辑”的开销，已经超出了直接加载这些寄存器数据的开销。这种情况下，可以选择禁用这个特性。

直接引入的开销，还包含了一些其他的。比如内核更新统计数据相关的，进程切换时，内核需要更新前一个任务执行的CPU时间；比如，内核需要执行调度算法选择下一个要执行的任务等等。

### 切换间接引入的开销

间接引入的开销，主要是与cache miss相关的，CPU本身有自己的L1、L2、L3级缓存，对于页式管理还有TLB（转换旁路缓冲），还有分支预测相关的（branch direction, branch target, return buffer），等等。



### 其他开销

[https://stackoverflow.com/a/54057079](https://stackoverflow.com/a/54057079)

## libgo patches get/setcontext

Google's Ian Lance Taylor explained in the commit improving libgo, "Currently, goroutine switches are implemented with libc getcontext/setcontext functions, which saves/restores the machine register states and also the signal context. This does more than what we need, and performs an expensive syscall. This CL implements a simplified version of getcontext/setcontext, in assembly, that only saves/restores the necessary part, i.e. the callee-save registers, and the PC, SP. A simplified version of makecontext, written in C, is also added. Currently this is only implemented on Linux/AMD64."

{% embed url="https://patchwork.ozlabs.org/project/gcc/patch/CAOyqgcUHhUkx0YqtLea1G5eFWg+0bQBJW-44V=jdjCJpW5VX4A@mail.gmail.com/" %}

## go cheaper context switch

#### [https://morioh.com/p/36af32e3f52c](https://morioh.com/p/36af32e3f52c) <a id="thread-context-switch"></a>

#### Thread Context Switch <a id="thread-context-switch"></a>

Interrupt processing, multi-tasking, user-mode switching, and other reasons will cause the CPU to switch from one thread to another thread. The switching process needs to save the state of the current process and restore the state of another.

The cost of context switching is high since swapping threads on the core takes a lot of time. The delay of context switching depends on different factors, probably between 50 and 100 nanoseconds. Considering that the hardware performs an average of 12 instructions per nanosecond on each core, then a context switch may take between 600 and 1200 instructions of latency. In fact, context switching takes up a lot of time for the program to execute instructions.

If there is a **Cross-Core Context Switch**, it may cause the CPU cache to fail. \(the cost of CPU accessing data from the cache is about 3 to 40 clock cycles, and the cost of accessing data from main memory is about 100 to 300 clock cycles \). The switching cost of this scenario will be more expensive.

### Go is born for concurrency <a id="go-is-born-for-concurrency"></a>

Since its official release in 2009, Golang has quickly gained market share due to its extremely high operating speed and efficient development efficiency. Golang supports concurrency from the language level and uses lightweight coroutines to implement concurrent programs.

Goroutine is very lightweight, mainly reflected in the following two aspects:

* The cost of context switching is low: Goroutine context switching involves only the modification of the value of three registers\(PC / SP / DX\) while the context switching of the comparison thread needs to include mode switching \(switching from user mode to kernel mode\) and 16 registers, PC, SP, etc. register refresh
* Less memory usage: thread stack space is usually 2M, Goroutine stack space is at least 2K;

Go programs can easily support six figures concurrent Goroutine operation, and when the number of threads reaches 1k, the memory consumption has reached 2G.

## measure the timecost of context switch

[https://blog.tsunanet.net/2010/11/how-long-does-it-take-to-make-context.html](https://blog.tsunanet.net/2010/11/how-long-does-it-take-to-make-context.html)



