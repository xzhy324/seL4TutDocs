# Notifications

2023.6.6 xzy



## Prerequisites

1. [Set up your machine](https://docs.sel4.systems/HostDependencies).
2. [Capabilities tutorial](https://docs.sel4.systems/Tutorials/capabilities)
3. [Mapping tutorial](https://docs.sel4.systems/Tutorials/mapping)
4. [Threads tutorial](https://docs.sel4.systems/Tutorials/threads)

## Outcomes

1. 了解如何在任务间构建共享内存区域
   在consumer端，将两段属于consumer的buffer映射到两个producer各自的虚拟地址空间（consumer端需要拿到producer vspace的cap）
2. 能够使用notification对象在不同任务间进行消息同步
3. 知道如何使用badges区分不同的信号量

## Background

notification机制允许进程之间异步的发送信号，主要用于中断处理以及同步访问共享数据缓冲区。

### ntfn对象的三态转换

有三个状态：

等待：有若干tcb在该notification上排队等待唤醒

活动：notification上有信号（不一定有接收对象）

空闲：没有TCB等待被唤醒，也没有TCB向其发消息



### seL4_Signal 调用 NBSend Syscall

若ntfn原本为空闲，则调用后转变为活动状态

若ntfn原本为活动，则调用后将调用cap的badge按位与到ntfn对象的数据字段上，同时若有线程在该ntfn上阻塞排队，唤醒队列头的线程

若ntfn原本为等待，则唤醒队列头

### seL4_Wait 调用Wait Syscall

若ntfn原本为空闲，则调用后转变为等待状态

若ntfn原本为活动，则调用后转变为空闲状态，且将notification的数据字段清空；

若ntfn原本为等待，则按照优先级将该TCB插入等待队列

### seL4_Polling 调用 NBWait Syscall

类似wait，但不加入tcb等待队列



## 问题

通过notification构造信号量，

一个可能的实现：

```c
typedef struct {
    seL4_Word count;
    seL4_CPtr notification;//相当于读写互斥锁
} semaphore_t;

void semaphore_init(semaphore_t *sem, seL4_Word initial ,seL4_CPtr ntfn) {
    sem->count = initial;
    //printf("inserting!!!\n");
    sem->notification = ntfn;
    /* error checking omitted */
    if(initial > 0) {
        seL4_Signal(sem->notification);
    }
}

void semaphore_wait(semaphore_t *sem, seL4_Word* badge) {
    //printf("waiting!!!\n");
    seL4_Wait(sem->notification, badge);
    sem->count--;
    printf("wait:count: %d\n", sem->count);
    if (sem->count > 0) {
        seL4_Signal(sem->notification);
    }
}

void semaphore_signal(semaphore_t *sem) {
    //printf("signaling!!!\n");
    sem->count++;
    if (sem->count >= 1) {
        seL4_Signal(sem->notification);
    }
}
```



## code answer

```c
#include <assert.h>
#include <sel4/sel4.h>
#include <stdio.h>
#include <utils/util.h>
#include <sel4utils/util.h>

// notification object
extern seL4_CPtr buf1_empty;
extern seL4_CPtr buf2_empty;
extern seL4_CPtr full;
extern seL4_CPtr endpoint;

extern seL4_CPtr buf1_frame_cap;
extern const char buf1_frame[4096];
extern seL4_CPtr buf2_frame_cap;
extern const char buf2_frame[4096];

// cslot containing a capability to the cnode of the server
extern seL4_CPtr consumer_vspace;
extern seL4_CPtr producer_1_vspace;
extern seL4_CPtr producer_2_vspace;

extern seL4_CPtr cnode;
extern seL4_CPtr mapping_1;
extern seL4_CPtr mapping_2;

#define BUF_VADDR 0x5FF000

int main(int c, char *argv[]) {
    seL4_Error error = seL4_NoError;
    seL4_Word badge;

    
    /* set up shared memory for consumer 1 */
    /* first duplicate the cap */
    error = seL4_CNode_Copy(cnode, mapping_1, seL4_WordBits, 
                          cnode, buf1_frame_cap, seL4_WordBits, seL4_AllRights);
    ZF_LOGF_IFERR(error, "Failed to copy cap");
    /* now do the mapping */
    error = seL4_ARCH_Page_Map(mapping_1, producer_1_vspace, BUF_VADDR, 
                               seL4_AllRights, seL4_ARCH_Default_VMAttributes);
    ZF_LOGF_IFERR(error, "Failed to map frame");
    
    // DONE share buf2_frame_cap with producer_2
    error = seL4_CNode_Copy(cnode, mapping_2, seL4_WordBits, 
                          cnode, buf2_frame_cap, seL4_WordBits, seL4_AllRights);
    error = seL4_ARCH_Page_Map(mapping_2, producer_2_vspace, BUF_VADDR, 
                               seL4_AllRights, seL4_ARCH_Default_VMAttributes);

    /* send IPCs with the buffer address to both producers */
    seL4_SetMR(0, BUF_VADDR);
    seL4_Send(endpoint, seL4_MessageInfo_new(0, 0, 0, 1));
    seL4_SetMR(0, BUF_VADDR);
    seL4_Send(endpoint, seL4_MessageInfo_new(0, 0, 0, 1));
    
    /* start single buffer producer consumer */
    volatile long *buf1 = (long *) buf1_frame;
    volatile long *buf2 = (long *) buf2_frame;

    *buf1 = 0;//置零表示的语义是当前缓冲区为空
    *buf2 = 0;

    
    // DONE signal both producers
    seL4_Signal(buf1_empty);
    seL4_Signal(buf2_empty);

    printf("Waiting for producer\n");

    for (int i = 0; i < 10; i++) {
        seL4_Wait(full, &badge);
        printf("Got badge: %lx\n", badge);
        
    // DONE, use the badge to check which producer has signalled you, and signal it back. Note that you 
    // may recieve more than 1 signal at a time.
    //注意badge是按位有效，一次性有多个signal时，会对当前notification的数据字段按位或，所以解析的时候要用按位与
        if (badge & 0b01) {
            printf("buf1: %ld\n", *buf1);
            seL4_Signal(buf1_empty);
            *buf1 = 0;
        }
        if (badge & 0b10) {
            printf("buf2: %ld\n", *buf2);
            seL4_Signal(buf2_empty);   
            *buf2 = 0;
        } 
   }

    printf("Success!\n");
    return 0;
}
```

