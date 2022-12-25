# Threads

2022.12.20 xzy



## Prerequisites

1. 需要安装haskell stack
2. ninja stack需要在线更新，若失败
   1.  检查是否能访问[这个网站](https://raw.githubusercontent.com/commercialhaskell/stackage-snapshots/master/lts/16/20.yaml)，若能，则配置shell代理在该网络环境下执行ninja
   2. 将stack换为国内的源，参考[stack换源](#Ref)

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



## Notes

### steps

1. allocate a untyped mem to tcb object
2. configure the TCB to have the same CSpace and VSpace as the current thread
3. Change priority via `seL4_TCB_SetPriority`
4. Set initial register state
5. Start the thread
6. Try to pass the arguments either by setting contexts or manipulating registers directly.
7. Resolving a fault

## API

### seL4_TCB_Configure

```
static inline int seL4_TCB_Configure
```

Set the parameters of a TCB

| Type         | Name               | Description                                                  |
| :----------- | :----------------- | :----------------------------------------------------------- |
| `seL4_TCB`   | `_service`         | Capability to the TCB which is being operated on.            |
| `seL4_Word`  | `fault_ep`         | CPTR to the endpoint which receives IPCs when this thread faults. This capability is in the CSpace of the thread being configured. |
| `seL4_CNode` | `cspace_root`      | The new CSpace root.                                         |
| `seL4_Word`  | `cspace_root_data` | Optionally set the guard and guard size of the new root CNode. If set to zero, this parameter has no effect. |
| `seL4_CPtr`  | `vspace_root`      | The new VSpace root.                                         |
| `seL4_Word`  | `vspace_root_data` | Has no effect on x86 or ARM processors.                      |
| `seL4_Word`  | `buffer`           | Location of the thread’s IPC buffer. Must be 512-byte aligned. The IPC buffer may not cross a page boundary. |
| `seL4_CPtr`  | `bufferFrame`      | Capability to a page containing the thread’s IPC buffer.     |



### Set Priority

```
static inline int seL4_TCB_SetPriority
```

Change a thread’s priority

| Type        | Name        | Description                                                  |
| :---------- | :---------- | :----------------------------------------------------------- |
| `seL4_TCB`  | `_service`  | Capability to the TCB which is being operated on.            |
| `seL4_TCB`  | `authority` | Capability to the TCB to use the MCP from when setting the priority. |
| `seL4_Word` | `priority`  | The thread’s new priority.                                   |



### Write Registers

```
static inline int seL4_TCB_WriteRegisters
```

Set a thread’s registers to the first `count` fields of a given seL4_UserContext

| Type                 | Name            | Description                                                  |
| :------------------- | :-------------- | :----------------------------------------------------------- |
| `seL4_TCB`           | `_service`      | Capability to the TCB which is being operated on.            |
| `seL4_Bool`          | `resume_target` | The invocation should also resume the destination thread.    |
| `seL4_Uint8`         | `arch_flags`    | Architecture dependent flags. These have no meaning on x86, ARM, and RISC-V. |
| `seL4_Word`          | `count`         | The number of registers to be set. **sizeof(regs)/sizeof(seL4_Word)** |
| `seL4_UserContext *` | `regs`          | Data structure containing the new register values.           |



## Code Answer

```c
//threads.c
#include <stdio.h>
#include <sel4/sel4.h>
#include <utils/util.h>
#include <sel4utils/util.h>
#include <sel4utils/helpers.h>


// the root CNode of the current thread
extern seL4_CPtr root_cnode;
// VSpace of the current thread
extern seL4_CPtr root_vspace;
// TCB of the current thread
extern seL4_CPtr root_tcb;
// Untyped object large enough to create a new TCB object

extern seL4_CPtr tcb_untyped;
extern seL4_CPtr buf2_frame_cap;
extern const char buf2_frame[4096];

// Empty slot for the new TCB object
extern seL4_CPtr tcb_cap_slot;
// Symbol for the IPC buffer mapping in the VSpace, and capability to the mapping
extern seL4_CPtr tcb_ipc_frame;
extern const char thread_ipc_buff_sym[4096];
// Symbol for the top of a 16 * 4KiB stack mapping, and capability to the mapping
extern const char tcb_stack_base[65536];
static const uintptr_t tcb_stack_top = (const uintptr_t)&tcb_stack_base + sizeof(tcb_stack_base);



int new_thread(void *arg1, void *arg2, void *arg3) {
    printf("Hello2: arg1 %p, arg2 %p, arg3 %p\n", arg1, arg2, arg3);
    void (*func)(int) = arg1;
    func(*(int *)arg2);
    while(1);
}

int my_func(int arg){
    printf("in my func: %d!\n",arg);
    return 0;
}
int my_data = 789;

int main(int c, char* arbv[]) {

    printf("Hello, World!\n");

    seL4_DebugDumpScheduler();

    // DONE fix the parameters in this invocation
    seL4_Error result = seL4_Untyped_Retype(tcb_untyped, seL4_TCBObject, seL4_TCBBits, root_cnode, 0, 0, tcb_cap_slot, 1);
    ZF_LOGF_IF(result, "Failed to retype thread: %d", result);
    seL4_DebugDumpScheduler();


    /* Exercise Now that you have a TCB object, configure it to have the same CSpace and VSpace 
     * as the current thread. Use the IPC buffer we have provided, but don't set a fault handler,
     * as the kernel will print any fault we receive with a debug build.
     */

    // DONE fix the parameters in this invocation
    result = seL4_TCB_Configure(tcb_cap_slot, seL4_CapNull, root_cnode, 0, root_vspace, 0, (seL4_Word)thread_ipc_buff_sym, tcb_ipc_frame);
    ZF_LOGF_IF(result, "Failed to configure thread: %d", result);

    // DONE fix the call to set priority using the authority of the current thread
    // and change the priority to 254
    result = seL4_TCB_SetPriority(tcb_cap_slot, root_tcb, 254);
    ZF_LOGF_IF(result, "Failed to set the priority for the new TCB object.\n");
    seL4_DebugDumpScheduler();

    seL4_UserContext regs = {0};
    int error = seL4_TCB_ReadRegisters(tcb_cap_slot, 0, 0, sizeof(regs)/sizeof(seL4_Word), &regs);
    ZF_LOGF_IFERR(error, "Failed to read the new thread's register set.\n");

    // DONE use valid instruction pointer
    sel4utils_set_instruction_pointer(&regs, (seL4_Word)&new_thread);
    // DONE use valid stack pointer
    sel4utils_set_stack_pointer(&regs, tcb_stack_top);
    // DONE fix parameters to this invocation

    sel4utils_arch_init_local_context((void*)new_thread,
                                  (void *)my_func, (void *)&my_data, (void *)3,
                                  (void *)tcb_stack_top, &regs);

    error = seL4_TCB_WriteRegisters(tcb_cap_slot, 0, 0, sizeof(regs)/sizeof(seL4_Word), &regs);
    ZF_LOGF_IFERR(error, "Failed to write the new thread's register set.\n"
                  "\tDid you write the correct number of registers? See arg4.\n");
    seL4_DebugDumpScheduler();

    // DONE resume the new thread
    error = seL4_TCB_Resume(tcb_cap_slot);
    ZF_LOGF_IFERR(error, "Failed to start new thread.\n");
    while(1);
    return 0;
}
```



## Ref 

https://docs.sel4.systems/projects/sel4/api-doc.html#map-5:~:text=inline%20int%20seL4_TCB_SetPriority-,Change%20a%20thread%E2%80%99s%20priority,-Type

[Scheduling domains [LWN.net\]](https://lwn.net/Articles/80911/)

[stack换源,注意hackage和stackage都需要更新](https://github.com/HaskellCNOrg/haskellcn/issues/171)
