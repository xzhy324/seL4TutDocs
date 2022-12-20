# Threads

2022.12.20 xzy



## Outcomes

1. Know the <u>jargon TCB</u>.
2. Learn how to start a thread in the same address space.
3. Understand how to read and update TCB register state.
4. Learn how to suspend and resume a thread.
5. Understand thread priorities and their interaction with the seL4 scheduler.
6. Gain a basic understanding of exceptions and debug fault handlers.

## Background

### Thread Control Blocks

seL4 provides threads to represent an execution context and manage processor time. Threads in seL4 are realised by *thread control block* objects (TCBs), one for each kernel thread.

TCBs contain the following information:

- a priority and maximum control priority,
- register state and floating-point context,
- CSpace capability,
- VSpace capability,
- endpoint capability to send fault messages to,
- and the reply capability slot.

### Scheduling model

The seL4 scheduler chooses the next thread to run on a specific processing core, and is a **priority-based FIFO round-robin scheduler**. The scheduler picks threads that are runnable: that is, resumed, and not blocked on any IPC operation.

#### Priorities

The scheduler picks the highest-priority, runnable thread. seL4 provides a priority range of 0-255, where 255 is the maximum priority (encoded in `libsel4` as `seL4_MinPrio` and `seL4_MaxPrio`).

TCBs also have a *maximum control priority* (MCP), which acts as an informal capability over priorities. When setting the priority of a TCB, an explicit TCB capability must be provided to derive the authority from to set the priority. The priority being set is checked against the authority TCB's MCP and the target priority is greater, the operation fails. The root task starts with both priority and MCP set to `seL4_MaxPrio`.

#### Round robin

When multiple TCBs are runnable and have the same priority, they are scheduled in a first-in, first-out round-robin fashion. In more detail, kernel time is accounted for in fixed-time quanta referred to as ticks, and each TCB has a timeslice field which represents the number of ticks that TCB is eligible to execute until preempted. The kernel timer driver is configured to fire a periodic interrupt which marks each tick, and when the timeslice is exhausted round robin scheduling is applied. Threads can surrender their current timeslice using the `seL4_Yield` system call.

#### [Domain scheduling](#Ref)

In order to provide confidentiality seL4 provides a top-level hierarchical scheduler which provides **static, cyclical scheduling of scheduling partitions known as domains**. Domains are statically configured at compile time with a cyclic schedule, and are non-preemptible resulting in completely deterministic scheduling of domains.

Threads can be assigned to domains, and threads are only scheduled when their domain is active. Cross-domain IPC is delayed until a domain switch, and seL4_Yield between domains is not possible. When there are no threads to run while a domain is scheduled, a domain-specific idle thread will run until a switch occurs.

Assigning a thread to a domain requires access to the `seL4_DomainSet` capability. This allows a thread to be added to any domain.

```c
/* Set thread's domain */
seL4_Error seL4_DomainSet_Set(seL4_DomainSet _service, seL4_Uint8 domain, seL4_TCB thread);
```

### Thread Attributes

seL4 threads are configured by [invocations on the TCB object](https://docs.sel4.systems/ApiDoc.html#sel4_tcb).

## Ref 

[Scheduling domains [LWN.net\]](https://lwn.net/Articles/80911/)

