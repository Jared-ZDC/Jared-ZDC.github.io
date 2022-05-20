---
layout: post
title: intel PAUSE指令功效分析
categories: 技术分析
description: pause指令的实现意义
keywords: pause intel 抢锁
---

# PAUSE指令
Improves the performance of spin-wait loops. When executing a “spin-wait loop,” processors will suffer a severe performance penalty when exiting the loop because it detects a possible memory order violation. The PAUSE instruction provides a hint to the processor that the code sequence is a spin-wait loop. The processor uses this hint to avoid the memory order violation in most situations, which greatly improves processor performance. For this reason, it is recommended that a PAUSE instruction be placed in all spin-wait loops.

An additional function of the PAUSE instruction is to reduce the power consumed by a processor while executing a spin loop. A processor can execute a spin-wait loop extremely quickly, causing the processor to consume a lot of power while it waits for the resource it is spinning on to become available. Inserting a pause instruction in a spin-wait loop greatly reduces the processor’s power consumption.

翻译成人话的意思是，其功效主要是两个：
* 提升抢锁性能
* 节省功耗
针对这两点，尝试使用一些testcase进行深入的测试分析，看看是否可以发现一些有意思的东西。

# 测试
之前在一个问题分析中有提到C库中Intel使用了rep nop（事实上就是PAUSE）来优化他的抢锁的性能，因此这里依然使用pthread_spin_lock的实现进行魔改来测试性能：

```c
//根据参数选择不同的spin_lock
if(pause_type)
        mypthread_spin_lock = mypthread_pause_spin_lock;
    else    
        mypthread_spin_lock = mypthread_nopause_spin_lock;


//测试主体 N个线程
void *doWork() {
    uint32_t loop = thread_cnt;
    uint32_t type = pause_type;
    uint64_t iloop = idle_loop;

    while(!readyFlag) ;

    while(loop--) {

        mypthread_spin_lock(&lockValue);
        loop_counter ++;
        mypthread_spin_unlock(&lockValue);
        iloop = idle_loop;
        while(iloop--);
    }
}
```
从实际测试结果上来看，性能似乎并没有太大变化，甚至于nopause的性能还要略优于pause的性能：
![image](https://user-images.githubusercontent.com/17999499/169471551-1a6f361f-fcd3-43c2-bfcd-78d6930946f4.png)


从测试功耗结果的角度看，确实节省了不少功耗，32个线程测试时，pause场景比nopaus场景，整机功耗能够下降将近25W左右，也就是说Intel手册上说明的功耗降低是有意义的，但是性能却没有体现出来。


# 分析
由此可以推断，在pthread_spin_lock的场景下，pause实际上是没有起到提升锁性能的作用，仅仅是起到了降低功耗的作用，那么pause在什么样的场景下 可以提升抢锁的性能？

在不同的架构下，pause 指令的延时其实是不同的，甚至于在icelake阶段，pause的延时已经达到了140个cycle了。至于为什么对抢锁性能没有较大的提升，这里可能有两个推论：
* 是否pause对原子的性能提升在实现上，更关注了CAS的性能，而忽略了FAA/SWAP的操作
* 是否pause的作用仅仅是提供了一个idle loop的作用，减少了冲突导致的性能提升？

修改pthread_spin_lock为cas实现方式，测试结果如下：
![image](https://user-images.githubusercontent.com/17999499/169471653-0d5b8664-9447-443d-b8e4-a436d4b8620a.png)


很有意思的是，发现性能也没有明显的变化，查看intel trm的文档，其中有一句话描述：
![image](https://user-images.githubusercontent.com/17999499/169471695-d52cea6b-bfd9-4ba6-a6a6-6a69cffd1ac4.png)


也就是说，这个并不是直接提升了原子指令的性能，而是一个hint给到处理器，告诉处理器当前处于spin_loop阶段，处理器的能力可以让度出去，避免了大多数场景里面了一些内存序冲突的问题，这里我的理解是减少了对其它共享内存操作的可能性，减少了核间竞争的的开销，提升了整体cache的有效性，从而业务性能有比较大的提升。

此外，在测试过程中，发现pause指令并不是单纯的执行nop指令，更多的像整个流水线都让渡出去了，从perf的执行统计上看，IPC非常低：
![image](https://user-images.githubusercontent.com/17999499/169471743-c609f809-fe5b-4a86-8460-1fb463bf9455.png)

也就是说 这个时候，基本另外的线程可以独占整个处理器，在性能上又有很大的提升。

## 进一步测试
在同一个物理核的超线程上面做redis的测试，确认IPC，在同物理核的另外一个线程上，运行测试程序，确认不同情况下，对redis的线程的影响：
nopause时， redis rps只能达到15万左右， 带了pause指令后， redis的性能可以达到16万吞吐量，这个还是计算并不密集的情况，比如在计算圆周率的场景下，pause指令对另外一个线程的吞吐量影响更大(IPC : 0.8->1.8)：
![image](https://user-images.githubusercontent.com/17999499/169471809-5b41ae67-99f4-442e-a2e6-b0848def60c8.png)


# 总结
* PAUSE指令并不是直接提升抢锁指令的性能，而是通过让渡CPU资源给同核心的另一个线程，提升整体运算利用率
* PAUSE指令在多线程的时候，还能减少对共享内存的访问，减少同cache的竞争，从而提升其它业务核的性能
* PAUSE指令能够极大的降低功耗


