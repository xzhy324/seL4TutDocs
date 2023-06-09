# Fault handling

2023.6.8

## Prerequisites

1. [Set up your machine](https://docs.sel4.systems/HostDependencies).
2. [Capabilities tutorial](https://docs.sel4.systems/Tutorials/capabilities).
3. [IPC Tutorial](https://docs.sel4.systems/Tutorials/ipc).

## Outcomes

1. 了解seL4定义的线程异常
2. 知道线程异常与通常意义上的处理器异常（如访问无效地址，除零）不同
3. 学习到fault handler是一个component中用于线程异常处理的特殊线程
4. 懂得当线程异常发生时，seL4内核会做什么
5. 了解如何设置内核用于异常处理的endpoint
6. 学习如何在线程发生异常后进行处理和恢复线程执行

## Thread faults vs other sources of faults

seL系统中异常的来源可以分为

- 处理器异常（例如访问无效内存地址，除零等等）
- 硬件异常（来自外设的异常），注意与硬件中断相区别
- **seL4内核产生的异常**，当目前运行线程出现错误时发生

本教程只关注第三种，并称其为线程异常

## How does thread fault handling work?

在seL4中，当一个线程产生线程异常的时候，内核将会阻塞该线程并尝试通过该线程一个特殊的ep fault handler（该线程tcb内存有这个ep cap的位置）发送异常消息。特殊的点在于每个线程只能拥有一个faulthandler ep，其余与普通ep无异。

在这个ep另一端监听的线程被称为fault handler，也就是在camkes在构建component时为每个component生成的一个特殊线程。当线程异常发生的时候，内核会唤醒错误处理线程并通过ep通知异常原因，错误处理线程需要处理本次异常，并在处理完成之后通知内核重新执行相应的线程。

通知内核恢复执行的方式有2：

1. 在fault handler ep上调用reply方法（IPC msginfo中的label字段应设置为0）
2. 通过 `seL4_TCB_Resume()`显式的告诉内核需要恢复执行

Please note that if the `handler` sets message registers in the reply message, the kernel may interpret these as meaning something: some fault replies accept parameters. See the seL4 manual for the reply message format for all faults.

如果错误处理线程没有处理错误而重新开始执行错误的线程，会导致错误的重复发生。

## Reasons for thread faults:

有多种原因可以导致线程错误，当错误发生时，内核会将错误原因通过IPC消息的方法发送给 fault handler。未开启MCS的内核的线程错误原因可以分为以下几类

- 权能异常: A fault triggered because of an invalid cap access.
- 虚拟内存异常: A fault triggered by incoherent（不一致的） page table state 【如果页表的状态不一致，可能是因为页表的某些部分在更新过程中没有正确地同步，或者是因为硬件故障导致的。这种情况下，系统可能无法正确地将虚拟地址转换为物理地址，从而导致VM fault（虚拟内存故障）。】or incorrect memory accesses by a thread.
- 未知的系统调用: Triggered by performing a syscall invocation that is unknown to the kernel.
- debug异常： Triggered when a breakpoint, watchpoint or single-step debug event occurs.

一个观察：上述异常都发生在内核空间，故发生后由内核捕获并通过ep发送给fh线程

在启用了MCS功能后，还增加

- 超时异常: Triggered when a thread consumes all of its budget and still has further execution to do in the current period.

## Thread fault messages:

当故障产生时，内核将通过该线程tcb上的fault ep发送一个IPC消息到fault handler线程。这个IPC消息包含

1. 故障发生的原因
2. 故障的周围上下文信息

每个异常都有其自己的消息格式，因为描述每个异常所需的信息将会不同。关于seL4内核为每个故障异常发送的IPC消息内容，请参阅[seL4手册](https://sel4.systems/Info/Docs/seL4-manual-latest.pdf)。



## Setting up a fault endpoint for a thread:

在一个fault通过fault ep被传送的情境中，内核作为IPC的“发送者”，而fault handler线程作为接收者。

这意味着当给fault endpoint分配caps时，需要给内核以及fault handler各自分配一个该ep对象的cap。

### Kernel end vs handler end:

在配置线程的TCB时，需要显式的声明该线程使用哪个fault handler。内核通过tcb拿到fault ep的cap的时候，为线程指定一个badge（采用mint方法生成带badge的ep cap），因此位于fault ep另一端的fault handler线程可以根据ipc消息中的badge来得到是哪个线程发生了异常，并从ipc消息的具体格式中解析出异常的原因和发生异常时的线程上下文。也即一个fault handler线程可以处理多个线程的异常。

### Differences between MCS and Master kernel:

There is a minor difference in the way that the kernel is informed of the cap to a fault endpoint, between the master and MCS kernels.

Regardless though, on both versions of the kernel, to inform the kernel of the fault endpoint for a thread, call the usual `seL4_TCB_SetSpace()`.

See the [MCS tutorial](https://docs.sel4.systems/Tutorials/mcs.html) for more information.





## Exercises

