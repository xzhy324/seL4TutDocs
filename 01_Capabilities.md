# Capabilities

2022.11.27 xzy

## Outcomes

By the end of this tutorial, you should be familiar with:

1. The jargon CNode, CSpace, CSlot.
2. Know how to invoke a capability.
3. Know how to delete and copy CSlots.

## Background

### What is a capability?

A *capability* is a unique, unforgeable token that gives the possessor permission to access an entity or object in system. One way to think of a capability is as a pointer with access rights. There are three kinds of capabilities in seL4:

- capabilities that control access to kernel objects such as thread control blocks,
- capabilities that control access to abstract resources such as `IRQControl`, and
- untyped capabilities, that are responsible for memory ranges and allocation from those (see also the [Untyped](https://docs.sel4.systems/Tutorials/untyped.html) tutorial).

In seL4, capabilities to all resources controlled by the kernel are given to the root task on initialisation. To change the state of any resource, user code can use the kernel API, available in `libsel4` to request an operation on the resource a specific capability points to.

For example, the root task is provided with a capability to its own thread control block (TCB), `seL4_CapInitThreadTCB`, a constant defined by `libsel4`. To change the properties of the initial TCB, one can use any of the [TCB API methods](https://docs.sel4.systems/projects/sel4/api-doc.html#sel4_tcb) on this capability.

Below is an example which changes the stack pointer of the root task's TCB, a common operation in the root task if a larger stack is needed:

```c
    seL4_UserContext registers;
    seL4_Word num_registers = sizeof(seL4_UserContext)/sizeof(seL4_Word);

    /* Read the registers of the TCB that the capability in seL4_CapInitThreadTCB grants access to. */
    seL4_Error error = seL4_TCB_ReadRegisters(seL4_CapInitThreadTCB, 0, 0, num_registers, &registers);
    assert(error == seL4_NoError);

    /* set new register values */
    registers.sp = new_sp; // the new stack pointer, derived by prior code.

    /* Write new values */
    error = seL4_TCB_WriteRegisters(seL4_CapInitThreadTCB, 0, 0, num_registers, &registers);
    assert(error == seL4_NoError);
```

The first argument of `seL4_TCB_ReadRegisters` and `seL4_TCB_WriteRegisters` addresses the capability in slot `seL4_CapInitThreadTCB`. We will explain addressing and slots below. The rest of the arguments are specific to the invocations. Further documentation is available on [TCB_ReadRegisters](https://docs.sel4.systems/projects/sel4/api-doc.html#read-registers) and [TCB_WriteRegisters](https://docs.sel4.systems/projects/sel4/api-doc.html#write-registers).

### CNodes and CSlots

A *CNode* (capability-node) is an object full of capabilities: you can think of a CNode as an array of capabilities. We refer to slots as *CSlots* (capability-slots). In the example above, `seL4_CapInitThreadTCB` is the slot in the root task's CNode that contains the capability to the root task's TCB. Each CSlot in a CNode can be in the following state:

- empty: the CNode slot contains a null capability,
- full: the slot contains a capability to a kernel resource.

By convention the 0th CSlot is kept empty, for the same reasons as keeping NULL unmapped in process virtual address spaces: to avoid errors when uninitialised slots are used unintentionally.

The field `info->CNodeSizeBits` gives a measure of the size of the initial CNode: it will have `1 << CNodeSizeBits` CSlots. A CSlot has `1 << seL4_SlotBits` bytes, so the size of a CNode in bytes is `1 << (CNodeSizeBits + seL4_SlotBits`.

### CSpaces

A *CSpace* (capability-space) is the full range of capabilities accessible to a thread, which may be formed of one or more CNodes. In this tutorial, we focus on the CSpace constructed for the root task by seL4's initialisation protocol, which consists of one CNode.

### CSpace addressing

To refer to a capability and perform operations on it, you must *address* the capability. There are two ways to address capabilities in the seL4 API. First is by invocation, the second is by direct addressing. Invocation is a shorthand and it is what we used to manipulate the registers of the root task's TCB, which we now explain in further detail.

#### Invocation

Each thread has a special CNode capability installed in its TCB as its *CSpace root*. This root can be empty (a null cap), for instance when the thread is not authorised to invoke any capabilities at all, or it can be a capability to a CNode. The root task always has a CSpace root capability that points to a CNode.

In an *invocation*, a CSlot is addressed by implicitly invoking the CSpace root of the thread that is performing the invocation. In the code example above, we use an invocation on the `seL4_CapInitThreadTCB` CSlot to read from and write to the registers of the TCB represented by the capability in that specific CSlot.

```c
seL4_TCB_WriteRegisters(seL4_CapInitThreadTCB, 0, 0, num_registers, &registers);
```

This implicitly looks up the `seL4_CapInitThreadTCB` CSlot in the CNode pointed to by the CSpace root capability of the calling thread, which here is the root task.

When it is clear by context and not otherwise relevant, we sometimes identify the capability with the object it points to. So, instead of saying "the CNode pointed to by the CSpace root capability", we sometimes say "the CSpace root". If not sure, it is better to be precise. Like structs and pointers in C, objects and capabilities are not actually interchangeable: one object can be pointed to by multiple capabilities, and each of these capabilities could have a different level of permissions to that object.

#### Direct CSpace addressing

In contrast to invocation addressing, *direct addressing* allows you to specify the CNode to look up in, rather than implicitly using the CSpace root. This form of addressing is primarily used to construct and manipulate the shape of CSpaces -- potentially the CSpace of another thread. Note that direct addressing also requires invocation: the operation occurs by invoking a CNode capability, which itself is indexed from the CSpace root.

The following fields are used when directly addressing CSlots:

- *_service/root* A capability to the CNode to operate on.
- *index* The index of the CSlot in the CNode to address.
- *depth* How far to traverse the CNode before resolving the CSlot.

For the initial, single-level CSpace, the *depth* value is always `seL4_WordBits`. For invocations, the depth is always implicitly `seL4_WordBits`. More on CSpace depth will be discussed in future tutorials.

In the example below, we directly address the root task's TCB to make a copy of it in the 0th slot in the CSpace root. [CNode copy](https://docs.sel4.systems/projects/sel4/api-doc.html#copy) requires two CSlots to be directly addressed: the destination CSlot, and the source CSlot. `seL4_CapInitThreadCNode` is used in three different roles here: as source root, as destination root, and as source slot. It is used as source and destination root, because we are copying within the same CNode -- the CNode of the initial thread. It is used as source slot, because within the CNode of the initial thread, `seL4_CapInitThreadCNode` is the slot of the capability we want to copy (and the slot `0` is the destination).

```c
    seL4_Error error = seL4_CNode_Copy(
        seL4_CapInitThreadCNode, 0, seL4_WordBits, // destination root, slot, and depth
        seL4_CapInitThreadCNode, seL4_CapInitThreadTCB, seL4_WordBits, // source root, slot, and depth
        seL4_AllRights);
    assert(error == seL4_NoError);
```

All [CNode invocations](https://docs.sel4.systems/projects/sel4/api-doc.html#sel4_cnode) require direct CSpace addressing.

### Initial CSpace

The root task has a CSpace, set up by seL4 during boot, which contains capabilities to all resources manages by seL4. We have already seen several capabilities in the root CSpace: `seL4_CapInitThreadTCB`, and `seL4_CapInitThreadCNode`. Both of these are specified by constants in `libsel4`, however not all initial capabilities are statically specified. Other capabilities are described by the `seL4_BootInfo` data structure, described in `libsel4` and initialised by seL4. `seL4_BootInfo` describes ranges of initial capabilities, including free slots available in the initial CSpace.

## Notes

#### Capability

Capability是用于提供访问系统中对象权限的凭证。seL4系统中所有资源的capability在启动时都被授予根进程。要对任何资源进行操作，用户都需使用libsel4中的内核API，并提供相应的capability。

##### CNode

CNode (capability-node) 是用于储存capability的对象，其中每个CSlot (capability-slot) 可为full或empty，分别对应是否拥有相应capability。CSlot的数量必须为2的整数次方倍。

> cslot对应于一个cap，一个cnode中拥有若干cslot（2^slotbits 个），一个cpace中可以有1个或多个cnode

##### CSpace

CSpace (capability-space) 是线程拥有的capability的集合，由一个或多个CNode组成。根进程的CSpace包含所有由seL4拥有资源的capability，在系统启动时被授予

**CSpace addressing**

To refer to a capability and perform operations on it, you must *address* the capability. There are two ways to address capabilities in the seL4 API. First is by invocation（由调用者自身隐式给出cap）, the second is by direct addressing（直接指定cap）.

## Code

```C

#include <stdio.h>
#include <sel4/sel4.h>
#include <sel4platsupport/bootinfo.h>
#include <utils/util.h>


int main(int argc, char *argv[]) {

    /* parse the location of the seL4_BootInfo data structure from
    the environment variables set up by the default crt0.S */
    seL4_BootInfo *info = platsupport_get_bootinfo();

    size_t initial_cnode_object_size = BIT(info->initThreadCNodeSizeBits);
    printf("Initial CNode is %zu slots in size\n", initial_cnode_object_size);


    size_t initial_cnode_object_size_bytes = initial_cnode_object_size * (1 << seL4_SlotBits); // DONE TODO calculate this.
    printf("The CNode is %zu bytes in size\n", initial_cnode_object_size_bytes);

    seL4_CPtr first_free_slot = info->empty.start;
    seL4_Error error = seL4_CNode_Copy(seL4_CapInitThreadCNode, first_free_slot, seL4_WordBits,
                                       seL4_CapInitThreadCNode, seL4_CapInitThreadTCB, seL4_WordBits,
                                       seL4_AllRights);
    ZF_LOGF_IF(error, "Failed to copy cap!");
    seL4_CPtr last_slot = info->empty.end - 1;
    /* DONE TODO use seL4_CNode_Copy to make another copy of the initial TCB capability to the last slot in the CSpace */
    error = seL4_CNode_Copy(seL4_CapInitThreadCNode, last_slot, seL4_WordBits,
                            seL4_CapInitThreadCNode, seL4_CapInitThreadTCB, seL4_WordBits,
                            seL4_AllRights);

    /* set the priority of the root task */
    error = seL4_TCB_SetPriority(last_slot, last_slot, 10);
    ZF_LOGF_IF(error, "Failed to set priority");

    // DONE TODO delete the created TCB capabilities
    error = seL4_CNode_Delete(seL4_CapInitThreadCNode, first_free_slot, seL4_WordBits);
    error = seL4_CNode_Delete(seL4_CapInitThreadCNode, last_slot, seL4_WordBits);
    //This can be done in either way: 
    //seL4_CNode_Delete: Delete a capability
    //seL4_CNode_Revoke: Delete all child capabilities of a capability
    //    =>seL4_CNode_Revoke(seL4_CapInitThreadCNode, seL4_CapInitThreadTCB, seL4_WordBits);

    // check first_free_slot is empty
    error = seL4_CNode_Move(seL4_CapInitThreadCNode, first_free_slot, seL4_WordBits,
                            seL4_CapInitThreadCNode, first_free_slot, seL4_WordBits);
    ZF_LOGF_IF(error != seL4_FailedLookup, "first_free_slot is not empty");

    // check last_slot is empty
    error = seL4_CNode_Move(seL4_CapInitThreadCNode, last_slot, seL4_WordBits,
                            seL4_CapInitThreadCNode, last_slot, seL4_WordBits);
    ZF_LOGF_IF(error != seL4_FailedLookup, "last_slot is not empty");

    printf("Suspending current thread\n");
    // DONE TODO suspend the current thread
    error = seL4_TCB_Suspend(seL4_CapInitThreadTCB);
    ZF_LOGF("Failed to suspend current thread\n");

    return 0;
}
```

## Ref

[Capabilities | seL4 docs](https://docs.sel4.systems/Tutorials/capabilities.html)

[x-qwq/feasibility.md at master · OSH-2019/x-qwq (github.com)](https://github.com/OSH-2019/x-qwq/blob/master/docs/feasibility.md) > 技术依据 > seL4内核机制

[API Reference | seL4 docs](https://docs.sel4.systems/projects/sel4/api-doc.html#copy)
