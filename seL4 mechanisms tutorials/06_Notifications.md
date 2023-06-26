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

一个可能的实现：[Show how counting semaphores can be implemented using only binary semaphores and ordinary machine instructions? - Stack Overflow](https://stackoverflow.com/questions/15117155/show-how-counting-semaphores-can-be-implemented-using-only-binary-semaphores-and/25220708#25220708)

```c
typedef struct {
    seL4_Word count;
    seL4_CPtr count_mutex;//相当于读写互斥锁
    seL4_CPtr critical_mutex;
} semaphore_t;

void semaphore_init(semaphore_t *sem, seL4_Word initial ,seL4_CPtr count_mutex,seL4_CPtr critical_mutex) {
    sem->count = initial;
    sem->count_mutex = count_mutex;
    seL4_CPtr critical_mutex = critical_mutex;
    seL_Signal(sem->count_mutex);
}

void semaphore_wait(semaphore_t *sem, seL4_Word* badge) {
    seL4_Wait(sem->count_mutex, NULL);
    sem->count--;
    if (sem->count < 0) {
        seL4_Signal(sem->count_mutex);	
        seL4_Wait(sem->critical_mutex, badge);
    } else {
        seL4_Signal(sem->count_mutex);
    }
}

void semaphore_signal(semaphore_t *sem) {
    seL4_Wait(sem->count_mutex, NULL);
    sem->count++;
    if (sem->count >= 0) {
        seL4_Signal(sem->count_mutex);
        seL4_Signal(sem->critical_mutex);
    } else {
        seL4_Signal(sem->count_mutex);
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
    //以当前进程的CNode权限，将得到的buf1_frame_cap复制到当前进程的CNode中，复制后的cap放在mapping_1这个slot中
    error = seL4_CNode_Copy(cnode, mapping_1, seL4_WordBits, 
                          cnode, buf1_frame_cap, seL4_WordBits, seL4_AllRights);
    ZF_LOGF_IFERR(error, "Failed to copy cap");
    /* now do the mapping */
    // 将mapping_1这个slot中的cap映射到一号生产者的vspace中的BUF_VADDR这个虚拟地址上
    error = seL4_ARCH_Page_Map(buf1_frame_cap, producer_1_vspace, BUF_VADDR, 
                               seL4_AllRights, seL4_ARCH_Default_VMAttributes);
    ZF_LOGF_IFERR(error, "Failed to map frame");
    
    // DONE share buf2_frame_cap with producer_2
    //重复上述工作即可
    error = seL4_CNode_Copy(cnode, mapping_2, seL4_WordBits, 
                          cnode, buf2_frame_cap, seL4_WordBits, seL4_AllRights);
    error = seL4_ARCH_Page_Map(mapping_2, producer_2_vspace, BUF_VADDR, 
                               seL4_AllRights, seL4_ARCH_Default_VMAttributes);

    /* send IPCs with the buffer address to both producers */
    // 利用IPC机制通知生产者映射的虚拟地址
    // 由于两个生产者都在同一个ep的另一端上排队，因此需要重复通知两次
    seL4_SetMR(0, BUF_VADDR);
    seL4_Send(endpoint, seL4_MessageInfo_new(0, 0, 0, 1));
    seL4_SetMR(0, BUF_VADDR);
    seL4_Send(endpoint, seL4_MessageInfo_new(0, 0, 0, 1));
    
    /* start single buffer producer consumer */
    //疑问：为什么可以直接访问这一段虚拟内存地址？是因为在camkes阶段已经映射到虚拟内存空间中了嘛？
    //解答： 这一段内存地址是位于consumer空间中的，当然可以直接访问
    volatile long *buf1 = (long *) buf1_frame;
    volatile long *buf2 = (long *) buf2_frame;

    *buf1 = 0;
    *buf2 = 0;

    
    // TODO signal both producers
    seL4_Signal(buf1_empty);
    seL4_Signal(buf2_empty);
    printf("Waiting for producer\n");

    //人为引入时间延迟
    int j=0;
    for (int i=0;i<100000;++i){
        j++;
    }   
    printf("j++:%d\n",j);

    for (int i = 0; i < 10; i++) {
        seL4_Wait(full, &badge);
        printf("Got badge: %lx\n", badge);
        
    // TODO, use the badge to check which producer has signalled you, and signal it back. Note that you 
    // may recieve more than 1 signal at a time.
        printf("Got badge: %lx\n", badge);
        if(badge & 0b01) {
            //printf("Got badge from producer 1\n");
            seL4_Signal(buf1_empty);
        }
        if (badge & 0b10) {
            //printf("Got badge from producer 2\n");
            seL4_Signal(buf2_empty);
        }
   }

    printf("Success!\n");
    return 0;
}
```

