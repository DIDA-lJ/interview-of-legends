## 说说强、软、弱、虚引用?

Java 根据其生命周期的长短将**引用类型**又分为强引用、软引用、弱引用、幻象引用。

- 强引用：就是我们平时 new 一个对象的引用。当 JVM 的内存空间不足时，宁愿抛出 OutOfMemoryError 使得程序异常终止，也不愿意回收具有强引用的存活着的对象。
- 软引用：生命周期比强引用短，当 JVM 认为内存空间不足时，会试图回收软引用指向的对象，也就是说在 JVM 抛出 OutOfMemoryError 之前，会去清理软引用对象，适合用在内存敏感的场景。
- 弱引用：比软引用还短，在 GC 的时候，不管内存空间足不足都会回收这个对象，ThreadLocal中的 key 就用到了弱引用，适合用在内存敏感的场景。
  -虚引用：也称幻象引用，之所以这样叫是因为虚引用的 get 永远都是 null ，称为get 了个寂寞，所以叫虚。

虚引用的唯一作用就是配合引用队列来监控引用的对象是否被加入到引用队列中，也就是可以准确的让我们知晓对象何时被回收。

还有一点有关虚引用的需要提一下，之前看文章都说虚引用对 gc 回收不会有任何的影响，但是看 1.8 doc 上面说

![](https://cdn.jsdelivr.net/gh/yessimida/cdn_image/img/image-20210228112643135.png)

> 简单翻译下就是：与软引用和弱引用不同，虚引用在排队时不会被垃圾回收器自动清除。通过**虚引用可访问的对象将保持这种状态，直到所有这些引用被清除或者它们本身变得不可访问**。

简单的说就是被虚引用引用的对象不能被 gc，然而在 JDK9 又有个变更记录：

![](https://cdn.jsdelivr.net/gh/yessimida/cdn_image/img/image-20210228112702582.png)

链接：https://bugs.openjdk.java.net/browse/JDK-8071507

按照这上面说的 JDK9 之前虚引用的对象是在虚引用自身被销毁之前是无法被 gc 的，而 JDK9 之后改了。

我没下 JDK9 ，不过我有 JDK11 ，所以看了下 11 doc 的确实改了。

![](https://cdn.jsdelivr.net/gh/yessimida/cdn_image/img/image-20210228112724158.png)

看起来是把那段删了。所以 JDK9 之前虚引用对引用对象的GC是有影响的，9及之后的版本没影响。

## 说说 Java 常见的垃圾收集器？

Java 虚拟机中有很多种垃圾收集器，下面简单介绍一些常用的：
### 1. Serial 收集器
Serial 收集器是最基础、历史最悠久的收集器，它是一个单线程收集器，在进行垃圾回收时，必须暂停其他所有的工作线程，直到收集结束，这是其主要缺点。

它的优点在于单线程避免了多线程复杂的上下文切换，因此在单线程环境下收集效率非常高，由于这个优点，迄今为止，其仍然是 HotSpot 虚拟机在客户端模式下默认的新生代收集器：


![Snipaste_2024-05-02_21-04-13.jpg](https://pic.code-nav.cn/mianshiya/question_picture/1783397053004488705/Snipaste_2024-05-02_21-04-13.jpg)

### 2. ParNew 收集器

它是 Serial 收集器的多线程版本，可以使用多条线程进行垃圾回收：


![Snipaste_2024-05-02_21-05-06.jpg](https://pic.code-nav.cn/mianshiya/question_picture/1783397053004488705/Snipaste_2024-05-02_21-05-06.jpg)

### 3. Parallel Scavenge 收集器

Parallel Scavenge 也是新生代收集器，基于 标记-复制 算法进行实现，它的目标是达到一个可控的吞吐量。这里的吞吐量指的是处理器运行用户代码的时间与处理器总消耗时间的比值：

```
吞吐量 = 运行用户代码时间 \ (运行用户代码时间 + 运行垃圾收集时间)
```

Parallel Scavenge 收集器提供两个参数用于精确控制吞吐量：

①、-XX:MaxGCPauseMillis：控制最大垃圾收集时间，假设需要回收的垃圾总量不变，那么降低垃圾收集的时间就会导致收集频率变高，所以需要将其设置为合适的值，不能一味减小。

②、-XX:MaxGCTimeRatio：直接用于设置吞吐量大小，它是一个大于 0 小于 100 的整数。假设把它设置为 19，表示此时允许的最大垃圾收集时间占总时间的 5%（即 1/(1+19) ）；默认值为 99 ，即允许最大 1%（ 1/(1+99) ）的垃圾收集时间。


### 4. Serial Old 收集器

从名字也能看出来，它是 Serial 收集器的老年代版本，同样是一个单线程收集器，采用 标记-整理 算法，主要用于给客户端模式下的 HotSpot 使用：


![Snipaste_2024-05-02_21-06-38.jpg](https://pic.code-nav.cn/mianshiya/question_picture/1783397053004488705/Snipaste_2024-05-02_21-06-38.jpg)

### 5. Paralled Old 收集器

Paralled Old 是 Parallel Scavenge 收集器的老年代版本，支持多线程并发收集，采用** 标记-整理** 算法实现：

![Snipaste_2024-05-02_21-08-38.jpg](https://pic.code-nav.cn/mianshiya/question_picture/1783397053004488705/Snipaste_2024-05-02_21-08-38.jpg)

### 6. CMS 收集器
CMS（Concurrent Mark Sweep）收集器是一种以获取最短回收停顿时间为目标的收集器，基于 标记-清除 算法实现，整个收集过程分为以下四个阶段：

1. 初始标记 (inital mark) ：标记 GC Roots 能直接关联到的对象，耗时短但需要暂停用户线程；
2. 并发标记 (concurrent mark) ：从 GC Roots 能直接关联到的对象开始遍历整个对象图，耗时长但不需要暂停用户线程；
3. 重新标记 (remark) ：采用增量更新算法，对并发标记阶段因为用户线程运行而产生变动的那部分对象进行重新标记，耗时比初始标记稍长且需要暂停用户线程；
4. 并发清除 (inital sweep) ：并发清除掉已经死亡的对象，耗时长但不需要暂停用户线程。


![Snipaste_2024-05-02_21-10-28.jpg](https://pic.code-nav.cn/mianshiya/question_picture/1783397053004488705/Snipaste_2024-05-02_21-10-28.jpg)

其优点在于耗时长的 并发标记 和 并发清除 阶段都不需要暂停用户线程，因此其停顿时间较短，其主要缺点如下：

- 由于涉及并发操作，因此对处理器资源比较敏感。
- 由于是基于 标记-清除 算法实现的，因此会产生大量空间碎片。
- 无法处理浮动垃圾（Floating Garbage）：由于并发清除时用户线程还是在继续，所以此时仍然会产生垃圾，这些垃圾就被称为浮动垃圾，只能等到下一次垃圾收集时再进行清理。

### 7. G1 收集器

G1 (Garbage-Frist) 收集器是一种面向服务器的垃圾收集器，主要应用在多核 CPU 和 大内存的服务器环境中。

G1 虽然也遵循分代收集理论，但不再以固定大小和固定数量来划分分代区域，而是把连续的 Java 堆划分为多个大小相等的独立区域（Region）。每一个 Region 都可以根据不同的需求来扮演新生代的 Eden 空间、Survivor 空间或者老年代空间，收集器会根据其扮演角色的不同而采用不同的收集策略。


![zongjie-e0f5da26-6e46-4f9d-bfcc-0842cc7079e7.png](https://pic.code-nav.cn/mianshiya/question_picture/1783397053004488705/zongjie-e0f5da26-6e46-4f9d-bfcc-0842cc7079e7.png)

上面还有一些 Region 使用 H 进行标注，它代表 Humongous，表示这些 Region 用于存储大对象（humongous object，H-obj），即大小大于等于 region 一半的对象。

G1 收集器的运行大致可以分为以下四个步骤：

①、初始标记 (Inital Marking) ：标记 GC Roots 能直接关联到的对象，并且修改 TAMS（Top at Mark Start）指针的值，让下一阶段用户线程并发运行时，能够正确的在 Reigin 中分配新对象。

G1 为每一个 Reigin 都设计了两个名为 TAMS 的指针，新分配的对象必须位于这两个指针位置以上，位于这两个指针位置以上的对象默认被隐式标记为存活的，不会纳入回收范围；

②、并发标记 (Concurrent Marking) ：从 GC Roots 能直接关联到的对象开始遍历整个对象图。遍历完成后，还需要处理 SATB 记录中变动的对象。

SATB（snapshot-at-the-beginning，开始阶段快照）能够有效的解决并发标记阶段因为用户线程运行而导致的对象变动，其效率比 CMS 重新标记阶段所使用的增量更新算法效率更高；

③、最终标记 (Final Marking) ：对用户线程做一个短暂的暂停，用于处理并发阶段结束后仍遗留下来的少量的 STAB 记录。虽然并发标记阶段会处理 SATB 记录，但由于处理时用户线程依然是运行中的，因此依然会有少量的变动，所以需要最终标记来处理；

④、筛选回收 (Live Data Counting and Evacuation) ：负责更新 Regin 统计数据，按照各个 Regin 的回收价值和成本进行排序，在根据用户期望的停顿时间进行来指定回收计划，可以选择任意多个 Regin 构成回收集。

然后将回收集中 Regin 的存活对象复制到空的 Regin 中，再清理掉整个旧的 Regin 。此时因为涉及到存活对象的移动，所以需要暂停用户线程，并由多个收集线程并行执行。


![Snipaste_2024-05-02_21-14-27.jpg](https://pic.code-nav.cn/mianshiya/question_picture/1783397053004488705/Snipaste_2024-05-02_21-14-27.jpg)

然后 ZGC 我写了一篇，可以看看呐。

[ZGC 的深入分析](https://mp.weixin.qq.com/s/5trCK-KlwikKO-R6kaTEAg)

## 垃圾回收，如何判断对象是否是垃圾？不同方式有什么区别？

一共有两种方式，分别是引用计数和可达性分析。

引用计数有循环依赖的问题，但是是可以解决的。

可达性分析则是从根引用（GCRoots） 开始进行引用链遍历扫描，如果可达则对象存活，如果不可达则对象已成为垃圾。

所谓的根引用包括全局变量、栈上引用、寄存器上的等。

![](https://cdn.jsdelivr.net/gh/yessimida/cdn_image/img/image-20210307095051363.png)



我之前也写过，**详细的看这篇文章吧，看完之后这一块针对面试绝对没问题，而且已经超越了很多面试官**。

[深度揭秘垃圾回收底层，这次让你彻底弄懂它](https://mp.weixin.qq.com/s/_wcTxJOCmZXS5MwHMNqbdQ)

## 你知道有哪些垃圾回收算法？

常见的就是：复制、标记-清除、标记整理。

### 标记-清除

标记-清除算法应该是最符合我们人一开始处理垃圾的思路的算法。

例如我们想清除房间的垃圾，我们肯定是先定位(对应标记)哪些是垃圾，然后把这些垃圾之后扔了(对应清除)，简单粗暴，剩下的不是垃圾的东西我也懒得理，不管了哈哈哈。

但是，这算法有个缺点：

1. 空间碎片问题，这样会使得比较大的对象要申请比较多的连续空间的时候申请不到，明明你空间还很足的。然后导致又一次GC。

### 复制算法

复制算法一般用于新生代，粗暴的复制算法就是把空间一分为二，然后将一边存活的对象复制到另一边，这样没有空间碎片问题，但是内存利用率太低了，只有 50%，所以 HotSpot 中是把一块空间分为 3 块，一块Eden,两块Survivor。

因为正常情况下新生代的大部分对象都是短命鬼，所以能活下来的不多，所以默认的空间划分比例是 8:1:1。

用法就是每次只使用Eden和一块Survivor,然后把活下来的对象都扔到另一块Survivor。再清理Eden和之前的那块Survivor。然后再把Eden和存放存活对象的那一块Survivor用来迎接新的对象。就等于每次回收了之后都会对调一下两个Survivor。

### 标记-整理算法

标记-整理算法的思路也是和标记-清除算法一样，先标记那些需要清除的对象，但是后续步骤不一样，它是整理，对就是像上面说的那些清除房间垃圾每次都会整理的人一样那么勤劳。

每次会移动所有存活的对象，且按照内存地址次序依次排列，也就是把活着的对象都像一端移动，然后将末端内存地址以后的内存全部回收。所以用了它也就没有空间碎片的问题了。

## young gc、old gc、full gc、mixed gc 的区别？

这个问题的前置条件是你得知道 GC 分代，为什么分代。这个在之前文章提了，不清楚的可以去看看。

现在我们来回答一下这个问题。

其实 GC 分为两大类，分别是 Partial GC 和 Full GC。

**Partial GC 即部分收集**，分为 young gc、old gc、mixed gc。

- young gc：指的是单单收集年轻代的 GC。
- old gc：指的是单单收集老年代的 GC。
- mixed gc：这个是 G1 收集器特有的，指的是收集整个年轻代和部分老年代的 GC。

**Full GC 即整堆回收**，指的是收取整个堆，包括年轻代、老年代，如果有永久代的话还包括永久代。

其实还有 Major GC 这个名词，在《深入理解Java虚拟机》中这个名词指代的是单单老年代的 GC，也就是和 old gc 等价的，不过也有很多资料认为其是和 full gc 等价的。

还有 Minor GC，其指的就是年轻代的 gc。

## young gc 触发条件是什么？

大致上可以认为在年轻代的 eden 快要被占满的时候会触发 young gc。

为什么要说大致上呢？因为有一些收集器的回收实现是在 full gc 前会让先执行以下 young gc。

比如 Parallel Scavenge，不过有参数可以调整让其不进行 young gc。

可能还有别的实现也有这种操作，不过正常情况下就当做 eden 区快满了即可。

eden 快满的触发因素有两个，一个是为对象分配内存不够，一个是为 TLAB 分配内存不够。

## full gc 触发条件有哪些？

这个触发条件稍微有点多，我们来看下。

- 在要进行 young gc 的时候，根据之前统计数据发现年轻代平均晋升大小比现在老年代剩余空间要大，那就会触发 full gc。
- 有永久代的话如果永久代满了也会触发 full gc。
- 老年代空间不足，大对象直接在老年代申请分配，如果此时老年代空间不足则会触发 full gc。
- 担保失败即 promotion failure，新生代的 to 区放不下从 eden 和 from 拷贝过来对象，或者新生代对象 gc 年龄到达阈值需要晋升这两种情况，老年代如果放不下的话都会触发 full gc。
- 执行 System.gc()、jmap -dump 等命令会触发 full gc。

## 知道 TLAB 吗？来说说看

这个得从内存申请说起。

一般而言生成对象需要向堆中的新生代申请内存空间，而堆又是全局共享的，像新生代内存又是规整的，是通过一个指针来划分的。

![](https://cdn.jsdelivr.net/gh/yessimida/cdn_image/img/20220123163413.png)

内存是紧凑的，新对象创建指针就右移对象大小 size 即可，这叫指针加法（bump [up] the pointer）。

可想而知如果多个线程都在分配对象，那么这个指针就会成为热点资源，需要互斥那分配的效率就低了。

于是搞了个 TLAB（Thread Local Allocation Buffer），为一个线程分配的内存申请区域。

**这个区域只允许这一个线程申请分配对象，允许所有线程访问这块内存区域**。

TLAB 的思想其实很简单，就是划一块区域给一个线程，这样每个线程只需要在自己的那亩地申请对象内存，不需要争抢热点指针。

当这块内存用完了之后再去申请即可。

这种思想其实很常见，比如分布式发号器，每次不会一个一个号的取，会取一批号，用完之后再去申请一批。

![](https://cdn.jsdelivr.net/gh/yessimida/cdn_image/img/20220123163511.png)

可以看到每个线程有自己的一块内存分配区域，短一点的箭头代表 TLAB 内部的分配指针。

如果这块区域用完了再去申请即可。

**不过每次申请的大小不固定**，会根据该线程启动到现在的历史信息来调整，比如这个线程一直在分配内存那么 TLAB 就大一些，如果这个线程基本上不会申请分配内存那 TLAB 就小一些。

还有 TLAB 会浪费空间，我们来看下这个图。

![](https://cdn.jsdelivr.net/gh/yessimida/cdn_image/img/20220123163528.png)

可以看到 TLAB 内部只剩一格大小，申请的对象需要两格，这时候需要再申请一块 TLAB ，之前的那一格就浪费了。

在 HotSpot 中会生成一个填充对象来填满这一块，**因为堆需要线性遍历**，遍历的流程是通过对象头得知对象的大小，然后跳过这个大小就能找到下一个对象，所以不能有空洞。

当然也可以通过空闲链表等外部记录方式来实现遍历。

**还有 TLAB 只能分配小对象，大的对象还是需要在共享的 eden 区分配**。

所以总的来说 TLAB 是为了避免对象分配时的竞争而设计的。

## 那 PLAB 知道吗？

可以看到和 TLAB 很像，PLAB 即 Promotion Local Allocation Buffers。

用在年轻代对象晋升到老年代时。

 在多线程并行执行 YGC 时，可能有很多对象需要晋升到老年代，此时老年代的指针就“热”起来了，于是搞了个 PLAB。

先从老年代 freelist（空闲链表） 申请一块空间，然后在这一块空间中就可以通过指针加法（bump the pointer）来分配内存，这样对 freelist 竞争也少了，分配空间也快了。

![](https://cdn.jsdelivr.net/gh/yessimida/cdn_image/img/20220123163550.png)

大致就是上图这么个思想，每个线程先申请一块作为 PLAB ，然后在这一块内存里面分配晋升的对象。

这和 TLAB 的思想相似。

## 产生 concurrent mode failure 真正的原因

> 《深入理解Java虚拟机》：由于CMS收集器无法处理“浮动垃圾”（FloatingGarbage），有可能出现“Con-current Mode Failure”失败进而导致另一次完全“Stop The World”的Full GC的产生。

这段话的意思是因为抛这个错而导致一次 Full GC。

而**实际上是 Full GC 导致抛这个错**，我们来看一下源码，版本是 openjdk-8。

首先搜一下这个错。

![](https://cdn.jsdelivr.net/gh/yessimida/cdn_image/img/20220123163605.png)

再找找看  `report_concurrent_mode_interruption` 被谁调用。

查到是在 `void CMSCollector::acquire_control_and_collect(...)` 这个方法中被调用的。

![](https://cdn.jsdelivr.net/gh/yessimida/cdn_image/img/20220123163623.png)

再来看看 first_state ： `CollectorState first_state = _collectorState;`

![](https://cdn.jsdelivr.net/gh/yessimida/cdn_image/img/20220123163634.png)

看枚举已经很清楚了，就是在 cms gc 还没结束的时候。

而 `acquire_control_and_collect` 这个方法是 cms 执行 foreground gc 的。

cms 分为  foreground gc 和 background gc。

foreground 其实就是 Full gc。

**因此是 full gc 的时候 cms gc 还在进行中导致抛这个错**。

究其原因是因为分配速率太快导致堆不够用，回收不过来因此产生 full gc。

也有可能是发起 cms gc 设置的堆的阈值太高。

## CMS GC 发生 concurrent mode failure 时的 full GC 为什么是单线程的?

**以下的回答来自 R 大**。

因为没足够开发资源，偷懒了。就这么简单。没有任何技术上的问题。 大公司都自己内部做了优化。

所以最初怎么会偷这个懒的呢？多灾多难的CMS GC经历了多次动荡。它最初是作为Sun Labs的Exact VM的低延迟GC而设计实现的。

但 Exact VM在与 HotSpot VM争抢 Sun 的正牌 JVM 的内部斗争中失利，CMS GC 后来就作为 Exact VM 的技术遗产被移植到了 HotSpot VM上。

就在这个移植还在进行中的时候，Sun 已经开始略显疲态；到 CMS GC 完全移植到 HotSpot VM 的时候，Sun 已经处于快要不行的阶段了。

开发资源减少，开发人员流失，当时的 HotSpot VM 开发组能够做的事情并不多，只能挑重要的来做。而这个时候 Sun Labs 的另一个 GC 实现，Garbage-First GC（G1 GC）已经面世。

相比可能在长时间运行后受碎片化影响的 CMS，G1 会增量式的整理/压缩堆里的数据，避免受碎片化影响，因而被认为更具潜力。

于是当时本来就不多的开发资源，一部分还投给了把G1 GC产品化的项目上——结果也是进展缓慢。

毕竟只有一两个人在做。所以当时就没能有足够开发资源去打磨 CMS GC 的各种配套设施的细节，配套的备份 full GC 的并行化也就耽搁了下来。

但肯定会有同学抱有疑问：HotSpot VM不是已经有并行GC了么？而且还有好几个？

让我们来看看：
- ParNew：并行的young gen GC，不负责收集old gen。
- Parallel GC（ParallelScavenge）：并行的young gen GC，与ParNew相似但不兼容；同样不负责收集old gen。
- ParallelOld GC（PSCompact）：并行的full GC，但与ParNew / CMS不兼容。

所以…就是这么一回事。

HotSpot VM 确实是已经有并行 GC 了，但两个是只负责在 young GC 时收集 young gen 的，这俩之中还只有 ParNew 能跟 CMS 搭配使用；

而并行 full GC 虽然有一个 ParallelOld，但却与 CMS GC 不兼容所以无法作为它的备份 full GC使用。

## 为什么有些新老年代的收集器不能组合使用比如 ParNew 和 Parallel Old？

![](https://cdn.jsdelivr.net/gh/yessimida/cdn_image/img/20220123163648.png)

这张图是 2008 年 HostSpot 一位 GC 组成员画的，那时候 G1 还没问世，在研发中，所以画了个问号在上面。

里面的回答是 :

> "ParNew" is written in a style... "Parallel Old" is not written in the "ParNew" style

HotSpot VM 自身的分代收集器实现有一套框架，只有在框架内的实现才能互相搭配使用。

而有个开发他不想按照这个框架实现，自己写了个，测试的成绩还不错后来被  HotSpot VM 给吸收了，这就导致了不兼容。

我之前看到一个回答解释的很形象：就像动车组车头带不了绿皮车厢一样，电气，挂钩啥的都不匹配。

## 新生代的 GC 如何避免全堆扫描？

在常见的分代 GC 中就是利用记忆集来实现的，记录可能存在的老年代中有新生代的引用的对象地址，来避免全堆扫描。

![](https://cdn.jsdelivr.net/gh/yessimida/cdn_image/img/20220123163657.png)

上图有个对象精度的，一个是卡精度的，卡精度的叫卡表。

把堆中分为很多块，每块 512 字节（卡页），用字节数组来中的一个元素来表示某一块，1表示脏块，里面存在跨代引用。

![](https://cdn.jsdelivr.net/gh/yessimida/cdn_image/img/20220123163707.png)

在 Hotspot 中的实现是卡表，是通过写后屏障维护的，伪代码如下。

![](https://cdn.jsdelivr.net/gh/yessimida/cdn_image/img/20220123163716.png)

cms 中需要记录老年代指向年轻代的引用，但是**写屏障的实现并没有做任何条件的过滤**。

即**不判断**当前对象是老年代对象且引用的是新生代对象才会标记对应的卡表为脏。

**只要是引用赋值都会把对象的卡标记为脏**，当然YGC扫描的时候只会扫老年代的卡表。

这样做是减少写屏障带来的消耗，毕竟引用的赋值非常的频繁。

## 那 cms 的记忆集和 G1 的记忆集有什么不一样？

cms 的记忆集的实现是卡表即 card table。

通常实现的记忆集是 points-out 的，我们知道**记忆集是用来记录非收集区域指向收集区域的跨代引用**，它的主语其实是非收集区域，所以是 points-out 的。

在 cms 中只有老年代指向年轻代的卡表，用于年轻代 gc。

而 G1 是基于 region 的，所以在 points-out 的卡表之上还加了个 points-into 的结构。

因为一个 region 需要知道**有哪些别的 region 有指向自己的指针，然后还需要知道这些指针在哪些 card 中**。

其实 G1 的记忆集就是个 hash table，key 就是别的 region 的起始地址，然后 value 是一个集合，里面存储这 card table 的 index。

我们来看下这个图就很清晰了。

![](https://cdn.jsdelivr.net/gh/yessimida/cdn_image/img/20220123163725.png)

像每次引用字段的赋值都需要维护记忆集开销很大，所以 G1 的实现利用了 logging write barrier（下文会介绍）。

也是异步思想，会先将修改记录到队列中，当队列超过一定阈值由后台线程取出遍历来更新记忆集。

## 为什么 G1 不维护年轻代到老年代的记忆集？

G1 分了 young GC 和 mixed gc。

young gc 会选取所有年轻代的 region 进行收集。

midex gc 会选取所有年轻代的 region 和一些收集收益高的老年代 region 进行收集。

所以**年轻代的 region 都在收集范围内，所以不需要额外记录年轻代到老年代的跨代引用**。

## cms 和 G1 为了维持并发的正确性分别用了什么手段？

之前文章分析到了并发执行漏标的两个充分必要条件是：

1. 将新对象插入已扫描完毕的对象中，即插入黑色对象到白色对象的引用。

2. 删除了灰色对象到白色对象的引用。

cms 和 g1 分别通过增量更新和 SATB 来打破这两个充分必要条件，维持了 GC 线程与应用线程并发的正确性。

cms 用了增量更新（Incremental update），打破了第一个条件，通过写屏障将插入的白色对象标记成灰色，即加入到标记栈中，在 remark 阶段再扫描，防止漏标情况。

G1 用了 SATB（snapshot-at-the-beginning），打破了第二个条件，会通过写屏障把旧的引用关系记下来，之后再把旧引用关系再扫描过。

这个从英文名词来看就已经很清晰了。讲白了就是在 GC 开始时候如果对象是存活的就认为其存活，等于拍了个快照。

而且 gc 过程中新分配的对象也都认为是活的。每个 region 会维持 TAMS （top at mark start）指针，分别是 prevTAMS 和 nextTAMS 分别标记两次并发标记开始时候 Top 指针的位置。

Top 指针就是 region 中最新分配对象的位置，所以 nextTAMS 和 Top 之间区域的对象都是新分配的对象都认为其是存活的即可。

![](https://cdn.jsdelivr.net/gh/yessimida/cdn_image/img/20220123163737.png)

而利用增量更新的 cms 在 remark 阶段需要重新所有线程栈和整个年轻代，因为等于之前的根有新增，所以需要重新扫描过，如果年轻代的对象很多的话会比较耗时。

要注意这阶段是 STW 的，很关键，所以 CMS 也提供了一个 CMSScavengeBeforeRemark 参数，来强制 remark 阶段之前来一次 YGC。

而 g1 通过 SATB 的话在最终标记阶段只需要扫描 SATB 记录的旧引用即可，从这方面来说会比 cms 快，但是也因为这样浮动垃圾会比 cms 多。

## 什么是 logging write barrier ？

写屏障其实耗的是应用程序的性能，是在引用赋值的时候执行的逻辑，这个操作非常的频繁，因此就搞了个 logging write barrier。

**把写屏障要执行的一些逻辑搬运到后台线程执行，来减轻对应用程序的影响**。

在写屏障里只需要记录一个 log 信息到一个队列中，然后别的后台线程会从队列中取出信息来完成后续的操作，其实就是异步思想。

像 SATB write barrier ，每个 Java 线程有一个独立的、定长的 SATBMarkQueue，在写屏障里只把旧引用压入该队列中。满了之后会加到全局 SATBMarkQueueSet。

![](https://cdn.jsdelivr.net/gh/yessimida/cdn_image/img/20220123163748.png)

后台线程会扫描，如果超过一定阈值就会处理，开始 tracing。

在维护记忆集的写屏障也用了 logging write barrier 。

## 简单说下 G1 回收流程

G1 从大局上看分为两大阶段，分别是并发标记和对象拷贝。

**并发标记**是基于 STAB 的，可以分为四大阶段：

1、初始标记（initial marking)，这个阶段是 STW 的，扫描根集合，标记根直接可达的对象即可。在G1中标记对象是利用外部的bitmap来记录，而不是对象头。

2、并发阶段（concurrent marking）,这个阶段和应用线程并发，从上一步标记的根直接可达对象开始进行 tracing，递归扫描所有可达对象。 STAB 也会在这个阶段记录着变更的引用。

3、最终标记（final marking), 这个阶段是 STW 的，处理 STAB 中的引用。

4、清理阶段（clenaup），这个阶段是 STW 的，根据标记的 bitmap 统计每个 region 存活对象的多少，如果有完全没存活的 region 则整体回收。

**对象拷贝阶段**（evacuation)，这个阶段是 STW 的。

根据标记结果选择合适的 reigon 组成收集集合（collection set 即 CSet），然后将 CSet 存活对象拷贝到新 region 中。 

G1 的瓶颈在于对象拷贝阶段，需要花较多的瓶颈来转移对象。

## 简单说下 cms 回收流程

其实从之前问题的 CollectorState 枚举可以得知几个流程了。

1、**初始标记(initial mark)**，这个阶段是 STW 的，扫描根集合，标记根直接可达的对象即可。

2、**并发标记(Concurrent marking)**，这个阶段和应用线程并发，从上一步标记的根直接可达对象开始进行 tracing，递归扫描所有可达对象。

3、**并发预清理(Concurrent precleaning)**，这个阶段和应用线程并发，就是想帮重新标记阶段先做点工作，扫描一下卡表脏的区域和新晋升到老年代的对象等，因为重新标记是 STW 的，所以分担一点。

4、**可中断的预清理阶段（AbortablePreclean）**，这个和上一个阶段基本上一致，就是为了分担重新标记标记的工作。

5、**重新标记(remark)**，这个阶段是 STW 的，因为并发阶段引用关系会发生变化，所以要重新遍历一遍新生代对象、Gc Roots、卡表等，来修正标记。

6、**并发清理(Concurrent sweeping)**，这个阶段和应用线程并发，用于清理垃圾。

7、**并发重置(Concurrent reset)**，这个阶段和应用线程并发，重置 cms 内部状态。

cms 的瓶颈就在于重新标记阶段，需要较长花费时间来进行重新扫描。

## cms 写屏障又是维护卡表，又得维护增量更新？

卡表其实只有一份，又得用来支持 YGC 又得支持 CMS 并发时的增量更新肯定是不够的。

每次 YGC 都会扫描重置卡表，这样增量更新的记录就被清理了。

所以**还搞了个 mod-union table**，在并发标记时，如果发生 YGC 需要重置卡表的记录时，就会更新  mod-union table 对应的位置。

这样 cms 重新标记阶段就能结合当时的卡表和  mod-union table 来处理增量更新，防止漏标对象了。

## GC 调优的两大目标是啥？

分别是**最短暂停时间和吞吐量**。

最短暂停时间：因为 GC 会 STW 暂停所有应用线程，这时候对于用户而言就等于卡顿了，因此对于时延敏感的应用来说减少 STW 的时间是关键。

吞吐量：对于一些对时延不敏感的应用比如一些后台计算应用来说，吞吐量是关注的重点，它们不关注每次 GC 停顿的时间，只关注总的停顿时间少，吞吐量高。

举个例子：

方案一：每次 GC 停顿 100 ms，每秒停顿 5 次。

方案二：每次 GC 停顿 200 ms，每秒停顿 2 次。

两个方案相对而言第一个时延低，第二个吞吐高，基本上两者不可兼得。

**所以调优时候需要明确应用的目标**。

## GC 如何调优

GC 调优这种问题肯定是具体场景具体分析，但是在面试中就不要讲太细，大方向说清楚就行，不需要涉及具体的垃圾收集器比如 CMS 调什么参数，G1 调什么参数之类的。

GC 调优的核心思路就是尽可能的使对象在年轻代被回收，减少对象进入老年代。

具体调优还是得看场景根据 GC 日志具体分析，常见的需要关注的指标是 Young GC  和 Full GC 触发频率、原因、晋升的速率、老年代内存占用量等等。

比如发现频繁会产生 Full GC，分析日志之后发现没有内存泄漏，只是 Young GC 之后会有大量的对象进入老年代，然后最终触发 Ful GC。所以就能得知是 Survivor 空间设置太小，导致对象过早进入老年代，因此调大 Survivor 。

或者是晋升年龄设置的太小，也有可能分析日志之后发现是内存泄漏、或者有第三方类库调用了 System.gc等等。

反正具体场景具体分析，核心思想就是尽量在新生代把对象给回收了。

基本上这样答就行了，然后就等着面试官延伸了。



---

TBC。

有更多相关的Java GC面试题，可以提PR哈，有错误欢迎联系我。

除了这个系列，我的公众号每周至少都会有一篇原创，欢迎关注~

![](https://gitee.com/yessimida/interview-of-legends/raw/master/pic/16034279-e6ebb79b5a0b8fe7.png)

最近已经汇集了近 500 名朋友交流各大小厂面试真题，也期待各位的面试题分享，公众号有我的联系方式，如果有兴趣可以加我备注 **面霸**，拉你进真题交流群。

![](https://cdn.jsdelivr.net/gh/yessimida/cdn_image/img/image-20210228190741512.png)
