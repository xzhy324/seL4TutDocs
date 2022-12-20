# Untyped

 2022.11.28 xzy

## Outcomes

1. The jargon *untyped*, *device untyped*, and *bit size*.
2. Know how to create objects from untyped memory in seL4.
3. Know how to reclaim objects.

一句话概括：熟悉seL4_Untyped_Retype() api的运行方式

## Background

### Physical memory

Apart from a small, static amount of kernel memory, all physical memory is managed by user level in an seL4 system. Capabilities to objects created by seL4 at boot, as well as the rest of the physical resources managed by seL4, are passed to the root task on start up.

### Untyped memory and capabilities

Excluding the objects used to create the root task, capabilities to all available physical memory are passed to the root task as capabilities to *untyped* memory. Untyped memory is a block of contiguous physical memory with a specific size. Untyped capabilities are capabilities to untyped memory. Untyped capabilities can be *retyped* into kernel objects together with capabilities to them, or into further, usually smaller, untyped capabilities.

Untyped capabilities have a boolean property *device* which indicates whether the memory is writable by the kernel or not: it may not be backed by RAM but some other device, or it may be in an area of RAM not addressable by the kernel. Device untyped capabilities can only be retyped intro frame objects (physical memory frames, which can be mapped into virtual memory), and cannot be written to by the kernel.

### Initial state

The `seL4_BootInfo` structure provided to the root task describes all of the untyped capabilities, including their size, if they are a device untyped, and the physical address of the untyped. The code example below shows how to print out the initial untyped capabilities provided from `seL4_BootInfo`.

```c
    printf("    CSlot   \tPaddr           \tSize\tType\n");
    for (seL4_CPtr slot = info->untyped.start; slot != info->untyped.end; slot++) {
        seL4_UntypedDesc *desc = &info->untypedList[slot - info->untyped.start];
        printf("%8p\t%16p\t2^%d\t%s\n", (void *) slot, (void *) desc->paddr, desc->sizeBits, desc->isDevice ? "device untyped" : "untyped");
    }
```

### Retyping

Untyped capabilities have a single invocation: [seL4_Untyped_Retype](https://docs.sel4.systems/ApiDoc.html#retype) which is used to create a new capability (and potentially object) from an untyped capability. Specifically, the new capability created by a retype invocation provides access to a subset of the memory range of the original capability, either as a smaller untyped capability, or pointing to a new object with a specific type. New capabilities created by retyping an untyped capability are referred to as *children* of that untyped capability.

Untyped capabilities are retyped incrementally in a greedy fashion from the invoked untyped, which is important to understand in order to gain efficient memory utilisation in seL4 systems. Each untyped capability maintains a single watermark, with addresses before the watermark being unavailable (already retyped) and after the watermark being free (not retyped yet). On a retype operation, the watermark is moved first to the alignment of the object being created, and then to the end of the size of the object. For example, if we create first a 4KiB object and then a 16KiB object, the 12KiB between the end of the 4KiB object and the start of the 16KiB object are wasted due to alignment. The memory cannot be reclaimed until both children are revoked.

TL;DR: objects should be allocated in order of size, largest first, to avoid wasting memory.

```c
    error = seL4_Untyped_Retype(parent_untyped, // the untyped capability to retype
                                seL4_UntypedObject, // type
                                untyped_size_bits,  //size
                                seL4_CapInitThreadCNode, // root
                                0, // node_index
                                0, // node_depth
                                child_untyped, // node_offset
                                1 // num_caps
                                );
```

The above code snippet shows an example of retyping an untyped into a smaller, 4KiB untyped capability (despite the name `seL4_UntypedObject` no object is created, the new untyped capability simply points to a subset of the untyped memory of `parent_untyped`. However, it is convenient to treat this memory like an object for the purposes of this operation). We will refer to this snippet as we discuss each parameter in the retype invocation. Recall that the first parameter is the untyped capability being invoked to do the retype.

#### Types

Each object in seL4 has a specific type, and all of the type constants can be found in `libsel4`. Some types are architecture specific, while others are general across architectures. In the example above, we create a new untyped capability, which can be further retyped. All other object type parameters create kernel objects *and* capabilities to them, and the type of the created object (and capability) determines the invocations that can be made on that capability. For example, were we to retype into a thread control block (TCB -- `seL4_TCBObject`), we could then perform TCB invocations on the capability to the new TCB object, including `seL4_TCB_SetPriority`.

#### Size

The size argument determines the size of the new object. The meaning of the argument depends on the object type that is being requested:

- most objects in seL4 are fixed size, and for these, the kernel will *ignore* the size argument;
- the types `seL4_UntypedObject` and `seL4_SchedContextObject` allow a variable size, which is specified in `size_bits` (more below);
- the type `seL4_CapTableObject` is also of variable size, and the argument specifies the *number of capability slots*. The same mechanism as for `size_bits` is used, but for the number of slots, not the size in bytes.

In general in seL4, if sizes are measured in bits, they are powers of two. "Bits" does not refer to the number of bits the object takes up, but the bit-width that is needed to describe its contents. An object of bit size `n` measures 2`n` bytes. Also, generally in seL4, objects and memory regions are aligned to their size, i.e. an `n`-bit object is aligned to 2`n` bytes, or, equivalently, the address of the object has 0 as its `n` bottom bits.

For `retype`, it is enough to remember that the parameter `size_bits` means the object will measure 2`size_bits` bytes, and for `seL4_CapTableObject` that you are requesting 2`size_bits` slots (you can compute the size in bytes by taking 2`size_bits + seL4_SlotBits`).

#### Root, node_index & node_depth

The `root`, `node_index` and `node_depth` parameters are used to specify the CNode in which to place the new capabilities. Depending on the `depth` parameter, the CSlot to use is addressed by invocation or by direct addressing (see the [capabilities tutorial](https://docs.sel4.systems/Tutorials/capabilities.html) for an explanation of those terms).

In the example above, `node_depth` is set to 0, which means invocation addressing is used: the `root` parameter is looked up implicitly using the CSpace root of the current thread at a depth of `seL4_WordBits`. So the example code specifies the root task's CNode (`seL4_CapInitThreadCNode`). The `node_index` parameter in this case is ignored.

If the `node_depth` value is not set to 0, then direct addressing is used with the current thread's CSpace root as the root. Then the `node_index` parameter is used to locate the CNode capability to place new capabilities in, at the specified `node_depth`. This is designed for managing multi-level CSpaces, and is not covered in this tutorial.

#### Node_offset

The node_offset is the CSlot to start creating new capabilities at, in the CNode selected by the previous parameters. In this case, the first empty CSlot in the initial CNode is selected.

#### Num_caps

The retype invocation can be used to create more than 1 capability and object at a time -- the number of capabilities is specified using this argument. Note that there are two constraints on this value:

1. The untyped must be big enough to fit all of the memory being retyped (`num_caps * (1u << size_bits)`).
2. The CNode must have enough consecutive free CSlots to fit all of the new capabilities.

## Notes

### seL4内存分配方式（retype）

按欲分配空间大小进行内存对齐之后，找到第一个满足大小的空闲块进行分配，且分配完之后，该object之前的所有内存地址均视为不可用

> eg 分配 4KB内存 再分配16KB内存，会导致16KBobject起始地址之前，4KBobject结束地址之后的12KB地址空间被打上水印标记为不可用

故而，分配内存时应按照largest object first的原则。

### seL4中的一些约定

**Bits**

In general in seL4, if sizes are measured in bits, they are powers of two. “Bits” does not refer to the number of bits the object takes up, but the bit-width that is needed to describe its contents. <u>An object of bit size `n` measures 2^`n` bytes</u>. Also, generally in seL4, <u>objects and memory regions are aligned to their size</u>, i.e. an `n`-bit object is aligned to 2`n` bytes, or, equivalently, the address of the object has 0 as its `n` bottom bits.

> 例子见Code line 36-41

### capabilities相关

The `root`, `node_index` and `node_depth` parameters are used to specify the CNode in which to place the new capabilities. Depending on the `depth` parameter, the CSlot to use is addressed by invocation or by direct addressing (see the [capabilities tutorial](https://docs.sel4.systems/Tutorials/capabilities.html) for an explanation of those terms).

```
In the example above, node_depth is set to 0, which means invocation addressing is used: the root parameter is looked up implicitly using the CSpace root of the current thread at a depth of seL4_WordBits. So the example code specifies the root task’s CNode (seL4_CapInitThreadCNode). The node_index parameter in this case is ignored.
If the node_depth value is not set to 0, then direct addressing is used with the current thread’s CSpace root as the root. Then the node_index parameter is used to locate the CNode capability to place new capabilities in, at the specified node_depth. This is designed for managing multi-level CSpaces, and is not covered in this tutorial.
depth指定为0，表示使用invocation adressing（cap默认为当前线程CSpace root，depth默认为seL4_WordBits）
depth指定为非0，表示使用direct addressing
注意这两种调用方式都是针对于seL4_Untyped_Retype()，比较的部分可以看下面的“区别一下”
```

> - *index* The index of the CSlot in the CNode to address.
> - *depth* How far to traverse the CNode before resolving the CSlot.



区别一下：

```c
seL4_CNode_Revoke(seL4_CapInitThreadCNode,//_service
                  child_untyped,//index (a cap or slot)
                  seL4_WordBits//depth (single-level will always be WordBits)
                 );
error = seL4_Untyped_Retype(parent_untyped, // the untyped capability to retype
                            seL4_UntypedObject, // type
                            untyped_size_bits,  //size
                            seL4_CapInitThreadCNode, // root
                            0, // node_index
                            0, // node_depth
                            child_untyped, // node_offset
                            1 // num_caps
                           );
```

中两个index 和 depth的不同。



## Enhancement

简单封装了一下对象分配，全部采用调用者空间隐式寻址，且一次只分配一个对象。



## Code

```c
#include <stdio.h>
#include <sel4/sel4.h>
#include <sel4platsupport/bootinfo.h>
#include <utils/util.h>

seL4_Error allocate_object(seL4_CPtr untyped,seL4_Word type,seL4_Word size_bits,seL4_CNode dest_root,seL4_CPtr dest_slot){
    return seL4_Untyped_Retype( untyped, // the untyped capability to retype
                                type, // type
                                size_bits,  //size
                                dest_root, // root
                                0, // node_index
                                0, // node_depth
                                dest_slot, // node_offset
                                1 // num_caps
                                );
}

int main(int argc, char *argv[]) {
    /* parse the location of the seL4_BootInfo data structure from
    the environment variables set up by the default crt0.S */
    seL4_BootInfo *info = platsupport_get_bootinfo();


    printf("    CSlot   \tPaddr           \tSize\tType\n");
    for (seL4_CPtr slot = info->untyped.start; slot != info->untyped.end; slot++) {
        seL4_UntypedDesc *desc = &info->untypedList[slot - info->untyped.start];
        printf("%8p\t%16p\t2^%d\t%s\n", (void *) slot, (void *) desc->paddr, desc->sizeBits, desc->isDevice ? "device untyped" : "untyped");
    }
    seL4_Error error;

    // list of general seL4 objects
    seL4_Word objects[] = {seL4_TCBObject, seL4_EndpointObject, seL4_NotificationObject};
    // list of general seL4 object size_bits
    seL4_Word sizes[] = {seL4_TCBBits, seL4_EndpointBits, seL4_NotificationBits};

    // DONE TODO work out what size object we need to create to be able to create all of the objects
    // listed above. Remember that sizes are in bits, that is, the exponents of powers of two.
    //注意到 一个tcb占11位，endpoint和notification占4和5位
    //因此分配12位空间 2^12 > 2^11 + 2^4 + 2^5 
    // wrong answer:seL4_Word untyped_size_bits = sizes[0] + sizes[1] + sizes[2];
    seL4_Word untyped_size_bits = seL4_TCBBits + 1; 
    seL4_CPtr parent_untyped = 0;
    seL4_CPtr child_untyped = info->empty.start;

    // First, find an untyped big enough to fit all of our objects
    for (int i = 0; i < (info->untyped.end - info->untyped.start); i++) {
        if (info->untypedList[i].sizeBits >= untyped_size_bits && !info->untypedList[i].isDevice) {
            parent_untyped = info->untyped.start + i;
            break;
        }
    }
    // create an untyped big enough to retype all of the above objects from

    error = seL4_Untyped_Retype(parent_untyped, // the untyped capability to retype
                                seL4_UntypedObject, // type
                                untyped_size_bits,  //size
                                seL4_CapInitThreadCNode, // root
                                0, // node_index
                                0, // node_depth
                                child_untyped, // node_offset
                                1 // num_caps
                                );
    ZF_LOGF_IF(error != seL4_NoError, "Failed to retype");

    // use the slot after child_untyped for the new TCB cap:
    seL4_CPtr child_tcb = child_untyped + 1;
    /* DONE TODO create a TCB in CSlot child_tcb */
    error = seL4_Untyped_Retype(child_untyped,
                                seL4_TCBObject,
                                seL4_TCBBits,//其实可以指定为0，因为TCBObject大小已知且固定，函数会忽略这个输入
                                seL4_CapInitThreadCNode,
                                0,
                                0,
                                child_tcb,
                                1
                                );
    // try to set the TCB priority
    error = seL4_TCB_SetPriority(child_tcb, seL4_CapInitThreadTCB, 10);
    ZF_LOGF_IF(error != seL4_NoError, "Failed to set priority");

    // use the slot after child_tcb for the new endpoint cap:
    seL4_CPtr child_ep = child_tcb + 1;
    /* DONE TODO create an endpoint in CSlot child_ep */
    
    // error = seL4_Untyped_Retype(
    //                             child_untyped,
    //                             seL4_EndpointObject,
    //                             seL4_EndpointBits,//同上
    //                             seL4_CapInitThreadCNode,
    //                             0,
    //                             0,
    //                             child_ep,
    //                             1
    //                             );
    
    error = allocate_object(child_untyped,seL4_EndpointObject,0,seL4_CapInitThreadCNode,child_ep);

    // identify the type of child_ep
    uint32_t cap_id = seL4_DebugCapIdentify(child_ep);
    ZF_LOGF_IF(cap_id == 0, "Endpoint cap is null cap");

    // use the slot after child_ep for the new notification cap:
    seL4_CPtr child_ntfn = child_ep + 1;
    // DONE TODO create a notification object in CSlot child_ntfn
    // error = seL4_Untyped_Retype(child_untyped,
    //                             seL4_NotificationObject,
    //                             seL4_NotificationBits,//同上
    //                             seL4_CapInitThreadCNode,
    //                             0,
    //                             0,
    //                             child_ntfn,
    //                             1
    //                             );
    error = allocate_object(child_untyped,seL4_NotificationObject,0,seL4_CapInitThreadCNode,child_ntfn);


    // try to use child_ntfn
    error = seL4_TCB_BindNotification(child_tcb, child_ntfn);
    ZF_LOGF_IF(error != seL4_NoError, "Failed to bind notification.");

    // DONE TODO revoke the child untyped
    //注意第一个是当前线程的CNode对象，第二个是要回收的slot或者说cap，第三个depth一般都是wordbits
    error = seL4_CNode_Revoke(seL4_CapInitThreadCNode,child_untyped,seL4_WordBits);

    // allocate the whole child_untyped as endpoints
    // Remember the sizes are exponents, so this computes 2^untyped_size_bits / 2^seL4_EndpointBits:
    seL4_Word num_eps = BIT(untyped_size_bits - seL4_EndpointBits);
    error = seL4_Untyped_Retype(child_untyped, seL4_EndpointObject, 0, seL4_CapInitThreadCNode, 0, 0, child_tcb, num_eps);
    ZF_LOGF_IF(error != seL4_NoError, "Failed to create endpoints.");

    printf("Success\n");

    return 0;
}

```

## REF

[Untyped | seL4 docs](https://docs.sel4.systems/Tutorials/untyped.html)



[the sel4 capability system.pdf]https://www.google.com/url?sa=t&rct=j&q=&esrc=s&source=web&cd=&ved=2ahUKEwjZiKeswtD7AhUwTWwGHb03A1sQFnoECBAQAQ&url=https%3A%2F%2Fwww.cl.cam.ac.uk%2Fresearch%2Fsecurity%2Fctsrd%2Fcheri%2Fworkshops%2Fpdfs%2F20160423-sel4-capabilities.pdf&usg=AOvVaw3RDnel9SnCig9r3eCM12rw







