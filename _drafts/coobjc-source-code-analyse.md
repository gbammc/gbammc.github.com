---
layout: post
title: "coobjc 源码分析"
keywords: ios,coobjc,promise,channel,context,coroutine
description: 通过 coobjc 理解协程
---

[coobjc](https://github.com/alibaba/coobjc) 是阿里新推出的 repo ，目的是为 iOS 开发带来协程功能。本文尝试分析其实现原理及梳理相关知识点。

## 协程

什么是协程？它能解决什么问题？首先根据[维基百科](https://zh.wikipedia.org/wiki/%E5%8D%8F%E7%A8%8B)的解释：

> 协程是计算机程序的一类组件，推广了非抢先多任务的子程序，允许执行被挂起与被恢复。相对子例程而言，协程更为一般和灵活，但在实践中使用没有子例程那样广泛。协程源自Simula和Modula-2语言，但也有其他语言支持。协程更适合于用来实现彼此熟悉的程序组件，如合作式多任务、异常处理、事件循环、迭代器、无限列表和管道。

协程本身是在一个线程里面执行，所以不管有多少个协程，它们都是串行运行的，也就是说同一时刻，在同一个线程里最多只有一个协程在运行。因此它避免了多线程编程可能导致的同步问题。

协程的行为有点像函数调用，但和函数调用不同的地方在于，协程支持挂起和恢复，也就是可以有多个入口和出口点。举个例子，对于协程来说，协程 A 可以切换到协程 B，然后协程 B 可以选择在某个点回到协程 A 的执行流，同时允许在另一个时刻，协程 A 重新回到协程 B 刚刚中断的地方，继续执行协程 B 后面的逻辑。这在函数中是不可能实现的，对于函数调用来说，函数 A 调用函数 B，必须等待函数 B 执行完毕后运行流程才会重新走回函数 A。

![coroutine_and_function]()

既然协程允许中途切换，以及支持再次从切换点进入继续执行，那就说明有相应的数据结构保存切换前后的上下文信息，在 coobjc 中这个数据结构就是: ```coroutine```

``` c
struct coroutine {
        coroutine_func entry;                   // Process entry.
        void *userdata;                         // Userdata.
        coroutine_func userdata_dispose;        // Userdata's dispose action.
        void *context;                          // Coroutine's Call stack data.
        void *pre_context;                      // Coroutine's source process's Call stack data.
        int status;                             // Coroutine's running status.
        uint32_t stack_size;                    // Coroutine's stack size
        void *stack_memory;                     // Coroutine's stack memory address.
        void *stack_top;                    // Coroutine's stack top address.
        struct coroutine_scheduler *scheduler;  // The pointer to the scheduler.
        int8_t   is_scheduler;                  // The coroutine is a scheduler.
        
        struct coroutine *prev;
        struct coroutine *next;
        
        void *autoreleasepage;                  // If enable autorelease, the custom autoreleasepage.
        bool is_cancelled;                      // The coroutine is cancelled
    };
```

对于程序运行，最重要的上下文就是当前寄存器和调用栈的状态，这些信息分别被保存在 ```context```，以及由 ```stack_size```，```stack_memory``` 和 ```stack_top``` 记录。

下面就来看 coobjc 的内核是如何利用这些信息实现协程的相关操作。

## coroutine_context

coobjc 的核心就在于以 ```coroutine_context``` 命名的这三个文件上。

![coroutine_context](https://i.loli.net/2019/06/22/5d0e3bce6a51193780.png)


因为 coobjc 的原理是模仿 ```ucontext.h``` 来实现上下文切换，所以 ```getcontext``` 和 ```setcontext``` 这两个重要的方法都在 arm64/armv7/x86_64/i386 这些架构上用汇编实现了一遍，统一暴露出来的接口是：

``` c
int coroutine_getcontext (coroutine_ucontext_t *__ucp);
int coroutine_setcontext (coroutine_ucontext_t *__ucp);
```

先看数据结构部分，上下文结构体 ```coroutine_ucontext_t``` 和寄存器结构体 ```coroutine_ucontext_re```，在 arm64 上的定义（后面的分析都是基于 arm64 架构进行）如下：

``` c
typedef struct coroutine_ucontext {
    uint64_t data[100];
} coroutine_ucontext_t;

struct coroutine_ucontext_re {
    struct GPRs {
        uint64_t __x[29]; // x0-x28
        uint64_t __fp;    // Frame pointer x29
        uint64_t __lr;    // Link register x30
        uint64_t __sp;    // Stack pointer x31
        uint64_t __pc;    // Program counter
        uint64_t padding; // 16-byte align, for cpsr
    } GR;
    double  VR[32];
};
```

arm64 跟其他架构不同，不仅有通用寄存器 34 个，还另有 32 个浮点寄存器专门用于浮点运算，所以 ```coroutine_ucontext_re``` 也分为了两部分，分别是 GR 和 VR，共 528（66 * 8）字节。 ```coroutine_ucontext_t``` 的大小也和别的架构不同，其它都是跟 ```coroutine_ucontext_re``` 一样大的，这里却是达到 800（100 * 8） 字节。

接着回到两个接口的实现上，虽然都是用汇编写的，但其实逻辑很简单，下面是 ```coroutine_getcontext``` 的源码：

``` s
.text
.align 2

/**
 Store the current calling stack(registers need to be store) into the memory passed by x0.
 
 This registers need to be saved:
 - x19-x28  Callee-saved registers.
 - x29,x30,sp  Known as fp,lr,sp.
 - d8-d15   Callee-saved vector registers.
 */
.global _coroutine_getcontext
_coroutine_getcontext:
    stp    x18,x19, [x0, #0x090]
    stp    x20,x21, [x0, #0x0A0]
    stp    x22,x23, [x0, #0x0B0]
    stp    x24,x25, [x0, #0x0C0]
    stp    x26,x27, [x0, #0x0D0]
    str    x28, [x0, #0x0E0];
    stp    x29, x30, [x0, #0x0E8];  // fp, lr
    mov    x9,      sp
    str    x9,      [x0, #0x0F8]
    str    x30,     [x0, #0x100]    // store return address as pc
    stp    d8, d9,  [x0, #0x150]
    stp    d10,d11, [x0, #0x160]
    stp    d12,d13, [x0, #0x170]
    stp    d14,d15, [x0, #0x180]
    mov    x0, #0                   
    ret
```

通过注释我们可以知道，```coroutine_getcontext``` 就是把需要的寄存器状态都保存起来，方法执行后的内存状态如下图所示：

![coroutine_getcontext](https://i.loli.net/2019/06/22/5d0e3d54bc46931728.png)


x0 的值是方法参数 ```__ucp ``` 的地址。其中 x30 既是 lr 也是作为恢复时的 pc 被储存了两次。可以看到这里只保存了 x19-x28、fp、lr、sp、pc，还有 d8-d15 这些寄存器。

保存的上下文都是为了在协程切换时能被恢复，```coroutine_setcontext``` 所做的就是把刚刚保存起来的寄存器都还原回去，同样的，逻辑也是很简单：

``` s
/**
 Restore the saved calling stack, and resume code at lr.
 */
.global _coroutine_setcontext
_coroutine_setcontext:
    ldp    x18,x19, [x0, #0x090]
    ldp    x20,x21, [x0, #0x0A0]
    ldp    x22,x23, [x0, #0x0B0]
    ldp    x24,x25, [x0, #0x0C0]
    ldp    x26,x27, [x0, #0x0D0]
    ldp    x28,x29, [x0, #0x0E0]
    ldr    x30,     [x0, #0x100]  // restore pc into lr
    ldr    x1,      [x0, #0x0F8]
    mov    sp,x1                  // restore sp
    ldp    d8, d9,  [x0, #0x150]
    ldp    d10,d11, [x0, #0x160]
    ldp    d12,d13, [x0, #0x170]
    ldp    d14,d15, [x0, #0x180]
    ldp    x0, x1,  [x0, #0x000]  // restore x0,x1
    ret    x30
```

ret 默认会使用 lr（x30） 的值作为 CPU 下条指令地址，但这里还是显式地调用了 ```ret    x30``` 一下，这样程序的执行就会恢复到之前调用 ```coroutine_getcontext``` 的位置，并接续执行后面的代码。

在 coroutine_context.h 里除了提供上面两个接口外，还有：

``` c
int coroutine_begin (coroutine_ucontext_t *__ucp);
void coroutine_makecontext (coroutine_ucontext_t *__ucp, IMP func, void *arg, void *stackTop);
```

```coroutine_begin``` 跟 ```coroutine_setcontext``` 一样，都是用于恢复上下文，唯一的区别是 ```coroutine_begin``` 会把 lr 设置为 0，这样会降低调用栈的深度，因此适用于协程的第一次调用。

```coroutine_makecontext``` 的作用是对协程上下文进行初始化，下文再详细解释。


基本的协程操作主要由下面两个函数完成：

核心的函数：

``` c
void coroutine_resume(coroutine_t *co);
void coroutine_yield(coroutine_t *co);
```

coroutine由用户协程和一个管理作用的协程组成。每次协程切换时，用户协程必须先切换到管理协程，然后管理协程再负责切换到其他的用户协程，不能直接从用户协程切换到其他用户协程。从以上两个函数来说，coroutine_resume就是从管理协程切换到用户协程的入口，coroutine_yield是从用户协程切换到管理协程的入口。

``` c
void coroutine_resume(coroutine_t *co) {
    if (!co->is_scheduler) {
        coroutine_scheduler_t *scheduler = coroutine_scheduler_self_create_if_not_exists();
        co->scheduler = scheduler;
        
        scheduler_queue_push(scheduler, co);
        
        if (scheduler->running_coroutine) {
            // resume a sub coroutine.
            scheduler_queue_push(scheduler, scheduler->running_coroutine);
            coroutine_yield(scheduler->running_coroutine);
        } else {
            // scheduler is idle
            coroutine_resume_im(co->scheduler->main_coroutine);
        }
    }
}
```

## Promise

## Actor Model

## CSP (Communicating Sequential Processes)

## References

[coobjc基本流程剖析](https://zhuanlan.zhihu.com/p/57917847)