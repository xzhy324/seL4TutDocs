# Interrupts

2023.6.7 xzy

## Prerequisites

1. [Set up your machine](https://docs.sel4.systems/HostDependencies).
2. [Notification tutorial](https://docs.sel4.systems/Tutorials/notifications)

## Outcomes

- 了解使用IRQControl cap的目的
- 能从给定的中断号获取相应的中断处理cap
- 学习如何处理中断，并了解中断处理和notification对象的关系。

## Background

### IRQControl

root task 被赋予了一个特殊的cap`seL4_CapIRQControl`，所有中断号对应的cap都从这个cap派生得到. 特殊之处在于，这个cap能够在root task不同Cspace和Cslot间移动但不能够被复制. 对这个cap进行revoke操作会回收对于所有派生的irq的访问cap.

### IRQHandlers

IRQHandler cap能够对某一个中断进行访问，这些handler是标准的cap：能够被移动和复制。

IRQHandler通过调用IRQControl cap获得，调用过程包含了体系结构相关的参数。以下是一个获取IRQHandler的例子：

```
// Get a capability for irq number 7 and place it in cslot 10 in a single-level cspace.
error = seL4_IRQControl_Get(seL4_IRQControl, 7, cspace_root, 10, seL4_WordBits);
```

以下是一些获得中断cap的方式，这些对于IRQControl的调用是体系结构相关的：

- [`seL4_IRQControl_GetIOAPIC`](https://docs.sel4.systems/ApiDoc.html#get-io-apic) (x86)
- [`seL4_IRQControl_GetMSI`](https://docs.sel4.systems/ApiDoc.html#get-msi) (x86)
- [`seL4_IRQControl_GetTrigger`](https://docs.sel4.systems/ApiDoc.html#gettrigger) (ARM)

### Receiving interrupts

接受一个中断是通过注册一个指向notification对象的cap实现的，irqhandler 会通过IPC cap transfer方式获得这个notification对象的cap：

```
seL4_IRQHandler_SetNotification(irq_handler, notification);
```

绑定成功后，当一个中断发生时，irqhandler会向notification发送一个信号。一个ntfn对象上可以绑定多个中断信号，通过ntfn的数据字段中的badge位来判断目前到来的是哪几个中断。

Interrupts can be polled for using `seL4_Poll` or waited for using `seL4_Wait`. Either system call results in the data word of the notification object being delivered as the badge of the message, and the data word cleared.

[`seL4_IRQHandler_Clear`](https://docs.sel4.systems/ApiDoc.html#clear) can be used to unbind the notification from an IRQHandler.

### Handling interrupts

当中断发生后，seL4会将对应的IRQHandler设置为masked屏蔽状态，此后该中断再次发生时，seL4不会响应这个中断。

因此，当中断发生并处理完成后，中断处理程序可以调用[`seL4_IRQHandler_Ack`](https://docs.sel4.systems/ApiDoc.html#ack) （ack for acknowledge 中断确认）来将该中断重新设置为非屏蔽状态。



## code answer

```c


#include <stdio.h>
#include <assert.h>
#include <sel4/sel4.h>
#include <timer_driver/driver.h>

// CSlots pre-initialised in this CSpace
extern seL4_CPtr endpoint;
// capability to a reply object
extern seL4_CPtr ntfn;
// capability to the device untyped for the timer

extern seL4_CPtr device_untyped;
// empty cslot for the frame
extern seL4_CPtr timer_frame;
// cnode of this process
extern seL4_CPtr cnode;
// vspace of this process
extern seL4_CPtr vspace;
// frame to map the timer to
extern seL4_CPtr frame;
extern const char timer_vaddr[4096];
// irq control capability
extern seL4_CPtr irq_control;
// empty slot for the irq
extern seL4_CPtr irq_handler;

/* constants */
#define EP_BADGE 61 // arbitrary (but unique) number for a badge
#define MSG_DATA 0x6161 // arbitrary data to send

#define DEFAULT_TIMER_ID 0
#define TTC0_TIMER1_IRQ 42

int main(void) {
    /* wait for a message */
    seL4_Word sender_badge;
    seL4_MessageInfo_t tag = seL4_Recv(endpoint, &sender_badge);

    /* make sure the message is what we expected */
    assert(sender_badge == EP_BADGE);
    assert(seL4_MessageInfo_get_length(tag) == 1);

    /* get the message stored in the first message register */
    seL4_Word msg = seL4_GetMR(0);
    printf("timer: got a message from %u to sleep %zu seconds\n", sender_badge, msg);

    /* retype the device untyped into a frame */
    seL4_Error error = seL4_Untyped_Retype(device_untyped, seL4_ARM_SmallPageObject, 0,
                                          cnode, 0, 0, timer_frame, 1);
    ZF_LOGF_IF(error, "Failed to retype device untyped");

    /* unmap the existing frame mapped at vaddr so we can map the timer here */
    error = seL4_ARM_Page_Unmap(frame);
    ZF_LOGF_IF(error, "Failed to unmap frame");

    /* map the device frame into the address space */
    error = seL4_ARM_Page_Map(timer_frame, vspace, (seL4_Word) timer_vaddr, seL4_AllRights, 0);
    ZF_LOGF_IF(error, "Failed to map device frame");

    timer_drv_t timer_drv = {0};

    
    /* DONE invoke irq_control to put the interrupt for TTC0_TIMER1_IRQ in
       cslot irq_handler (depth is seL4_WordBits) */
    error = seL4_IRQControl_Get(irq_control, TTC0_TIMER1_IRQ, cnode, irq_handler, seL4_WordBits);
     /* DONE set ntfn as the notification for irq_handler */
    error = seL4_IRQHandler_SetNotification(irq_handler, ntfn);
    /* set up the timer driver */
    int timer_err = timer_init(&timer_drv, DEFAULT_TIMER_ID, (void *) timer_vaddr);
    ZF_LOGF_IF(timer_err, "Failed to init timer");

    timer_err = timer_start(&timer_drv);
    ZF_LOGF_IF(timer_err, "Failed to start timer");

    /* ack the irq in case of any pending interrupts int the driver */
    error = seL4_IRQHandler_Ack(irq_handler);
    ZF_LOGF_IF(error, "Failed to ack irq");

    timer_err = timer_set_timeout(&timer_drv, NS_IN_MS, true);
    ZF_LOGF_IF(timer_err, "Failed to set timeout");

    int count = 0;
    while (1) {
        /* Handle the timer interrupt */
        seL4_Word badge;
        seL4_Wait(ntfn, &badge);
        timer_handle_irq(&timer_drv);
        if (count == 0) {
            printf("Tick\n");
        }
        
        /* DONE ack the interrupt */
        error = seL4_IRQHandler_Ack(irq_handler);
        count++;
        if (count == 1000 * msg) {
            break;
        }
    }

    // stop the timer
    timer_stop(&timer_drv);

   /* modify the message */
    seL4_SetMR(0, 0);

    /* send the modified message back */
    seL4_ReplyRecv(endpoint, tag, &sender_badge);

    return 0;
}
```

