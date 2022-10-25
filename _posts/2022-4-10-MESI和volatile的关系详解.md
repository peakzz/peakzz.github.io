---
layout:     post
title:      "MESI和volatile的关系详解"
subtitle:   "volatile将MESI退化为强一致性"
date:       2022-4-10 10:00:00
author:     "Peakiz"
header-img: "img/post-bg-cpu.png"
header-mask: 0.30
catalog: true
tags:
    - java基础
    - 并发安全
---

### 什么是MESI协议？
现代CPU多核架构中为了协调快速的CPU运算和相对较慢的内存读写速度之间的矛盾，在CPU和内存之间引入了CPU cache：

<img src="https://raw.githubusercontent.com/peakzz/PicturesBed/master/img/2022/10/24/-234938.png" alt="image-20221024234936866" style="zoom:40%;" />

上图就是简化版的多核CPU架构图，数据先从 Memory 读取到 Cache 中，然后每次 CPU 都从 Cache 中取数据，CPU 如果要修改数据，也是先将数据写入 Cache，然后再由 Cache 刷新至 Memory 中，CPU 以 cache line（缓存行）为读写单位，即 Memory 与 Cache 数据交换的最小单元为 cache line。

java中并发的线程都分配在各个 CPU 上执行，每个线程都有自己的工作内存，也就相当于 CPU 中的 Cache，共享变量的副本其实是在 cache line 中的。

引入了cache就同时也带来了缓存一致性问题：**如何让CPU及时看到其他CPU核心修改后的数据**；于是为了保证 CPU 间缓存数据的一致性，科学家们引入了缓存一致性协议，比较常用的缓存一致性协议为 MESI 协议。MESI协议是一个基于失效的缓存一致性协议，是支持写回（write-back）缓存的最常用协议。

MESI协议下，缓存行(cache line)有四种状态，运行过程中数据在这四种状态中进行流转：
- 已修改Modified (M)
  缓存行是脏的，与主存的值不同。如果别的CPU内核要读主存这块数据，该缓存行必须回写到主存，状态变为共享(S)
  
- 独占Exclusive (E)
  缓存行只在当前缓存中，但是干净的（clean）--缓存数据同于主存数据。当别的缓存读取它时，状态变为共享；当前写数据时，变为已修改状态。
  
- 共享Shared (S)
  缓存行也存在于其它缓存中且是干净的。缓存行可以在任意时刻抛弃。
  
- 无效Invalid (I)
  缓存行是无效的，需要从主内存中读取最新值；

**严格的MESI协议是同步数据更新的！完全解决了缓存一致性问题；**

<img src="https://raw.githubusercontent.com/peakzz/PicturesBed/master/img/2022/10/25/-203356.png" alt="image-20221025203355347" style="zoom:33%;"  align="cneter"/>

<center>对MESI协议的优化</center>

如上图所示：

- 当读取数据时：cpu会在总线上广播读请求，别的cpu收到读请求后会将自己缓存中被修改了和没有刷新进入主内存的数据(M状态)进行刷新；
- 当cpu1写入数据时：数据状态为s的时候先会往总线上广播一条 invalidate 消息，其他 CPU 收到消息后，会将其缓存行置为 I，然后发一个 invalidate ack 消息给 CPU1，CPU1收到此消息后会将 a=2 写入缓存行中，然后再同步到内存，最后会将缓存行状态置为 E（因为其它 CPU 缓存行都失效了，所以缓存行为此 CPU 独有）

> 对mesi的改进

如果 CPU 之间严格遵循 MESI 协议，那其实也没 volatile 什么事了，但问题是如果严格遵循 MESI 协议的话，CPU 的执行效率会受到严重影响，因为每次要修改缓存，如果缓存行状态为 S 的话都要先发一个 invalidate 的广播，再等其他 CPU 将缓存行设置为无效后返回 invalidate ack 才能写到 Cache 中，那如果 CPU 频繁地修改数据，就会不断地发送广播消息，CPU 只能被动同步地等待其他 CPU 的消息，显然会对执行效率产生影响。为了解决此问题，工程师在 CPU 和 cache 之间又加了一个 store buffer，同时在cache和总线之间添加了Invalidate Queue；

<img src="https://raw.githubusercontent.com/peakzz/PicturesBed/master/img/2022/10/25/-203737.png" alt="image-20221025203736036" style="zoom:33%;" align="center"/>

 CPU 要修改数据，先写入 store buffer 中，然后马上返回，之后再由 store buffer 异步执行发送广播消息和写入 cache 的操作；因此store buffer 主要是用来将 invalidate 广播消息异步化处理；

CPU 收到 invalidate 广播消息，先将此消息存储在 invalidate queue 中，然后立即回复 invalidate ack 消息给发出广播的那个 CPU，之后 invalidate queue 再异步执行将缓存行失效（设置状态为 I）的操作；因此invalidate Queue是将缓存失效行为异步化；

从 store buffer 发送 invalidate 广播到 invalidate queue 让缓存行失效期间，CPU 可能读取到本应该失效的cache数据，导致数据不一致。**store buffer和invalidate Queue的加入使得MESI协议的强一致性变为了最终一致性.**

### volatile和MESI协议之间的关系？

结论先行：**volatile 是CPU硬件工程师给程序员留的一个口子，把对MESI协议的优化(store buffer, invalidate queue)禁用，暂时以同步方式工作，使得对于该关键字的MESI协议退回强一致性状态！**

volatile 用来告诉CPU对这个变量的读写不要瞎优化， 不要乱序、也不要流水线、storeBuffer、invalidateQueue 也暂时用同步的方式来工作， 这个才能保证变量的读写结果和程序员的预期一致。volatile变量的读写效率肯定比普通变量低， 但是多线程代码的首要目标是要保证正确性， 其次才是性能。所以， 代码设计时， 多线程之间能不共享变量就不共享；

>  那么volatile 怎么实现对特定变量禁用异步优化的呢？

通过lock前缀指令实现的。用 volatile 修饰的变量，在编译成机器指令时会在**写操作**后面，加上一条特殊的指令：`“lock addl #0x0, (%rsp)”`，这条指令会将 CPU 对此变量的修改，**立即写入内存，并通知其他 CPU 将缓存行置为无效状态***。**相当于将store buffer, invalidate queue禁用了！**

lock前缀是一个特殊的信号，是用来锁总线的，上锁时禁止总线及缓存的读写操作，执行过程如下：

- 对总线和缓存上锁。

- 强制所有lock信号之前的指令，都在此之前被执行，并同步相关缓存。

  

>  volatile除了使变量具有可见性外，还有一个功能就是禁用指令重排，这个功能就和MESI没什么太大关系了，那么禁止指令重排是怎么实现的呢？

利用读写内存屏障实现的。

现代 CPU 普遍是是采用流水线机制动作的，为了提升运行效率和提高缓存命中率，会采用乱序执行。

被 volatile 修饰的变量其实就相当于在**变量写**的时候添加了**写屏障**，避免了 volatile 变量写与在其**之前**其它变量写的排序，在变量读的时候添加了读屏障，避免了 volatile 读与在其**之后**的变量读的排序。

volatile 修饰的变量遵循happens before 规则，之前指令对它的操作结果一定会被后面的操作指令所感知到；

假设cpu写cache都是按照指令顺序fifo写的，那现在可以抛弃volatile吗？对于arm和power这个weak consistency的架构的cpu来说，它们只会保证指令之间有比如控制依赖，数据依赖，地址依赖等等依赖关系的指令间提交的先后顺序，而对于完全没有依赖关系的指令，比如x=1;y=2，它们是不会保证执行提交的顺序的，除非你使用了volatile，java把volatile编译成arm和power能够识别的barrier指令，这个时候才是按顺序的。



#### 参考：

1. [既然CPU有缓存一致性协议（MESI），为什么JMM还需要volatile关键字？ - 知乎 (zhihu.com)](https://www.zhihu.com/question/296949412/answer/2709967017?utm_source=qq&utm_medium=social&utm_oi=795400168655720448)







 