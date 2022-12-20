# Capabilities

完成日期：2022.11.27

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