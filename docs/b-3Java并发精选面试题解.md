## 如果让你设计一个线程池如何设计？

这种设计类问题还是一样，先说下理解，表明你是知道这个东西的用处和原理的，然后开始 BB。基本上就是按照现有的设计来说，再添加一些个人见解。

线程池讲白了就是存储线程的一个容器，池内保存之前建立过的线程来重复执行任务，减少创建和销毁线程的开销，提高任务的响应速度，并便于线程的管理。

我个人觉得如果要设计一个线程池的话得考虑池内工作线程的管理、任务编排执行、线程池超负荷处理方案、监控。

初始化线程数、核心线程数、最大线程池都暴露出来可配置，包括超过核心线程数的线程空闲消亡配置。

任务的存储结构可配置，可以是无界队列也可以是有界队列，也可以根据配置分多个队列来分配不同优先级的任务，也可以采用 stealing 的机制来提高线程的利用率。

再提供配置来表明此线程池是 IO 密集还是 CPU 密集型来改变任务的执行策略。

超负荷的方案可以有多种，包括丢弃任务、拒绝任务并抛出异常、丢弃最旧的任务或自定义等等。

线程池埋好点暴露出用于监控的接口，如已处理任务数、待处理任务数、正在运行的线程数、拒绝的任务数等等信息。

我觉得基本上这样答就差不多了，等着面试官的追问就好。

注意不需要跟面试官解释什么叫核心线程数之类的，都懂的没必要。

当然这种开放型问题还是仁者见仁智者见智，我这个不是标准答案，仅供参考。

## 并发类库提供的线程池实现有哪些?

虽说阿里巴巴Java 开发手册禁止使用这些实现来创建线程池，但是这问题我被问过好几次，也是热点。

问着问着就会延伸到线程池是怎么设计的。

> 我先来说下线程池的内部逻辑，这样才能理解这几个实现。

首先线程池有几个关键的配置：核心线程数、最大线程数、空闲存活时间、工作队列、拒绝策略。

1. 默认情况下线程不会预创建，所以是来任务之后才会创建线程（设置prestartAllCoreThreads可以预创建核心线程）。
2. 当核心线程满了之后不会新建线程，而是把任务堆积到工作队列中。
3. 如果工作队列放不下了，然后才会新增线程，直至达到最大线程数。
4. 如果工作队列满了，然后也已经达到最大线程数了，这时候来任务会执行拒绝策略。
5. 如果线程空闲时间超过空闲存活时间，并且线程线程数是大于核心线程数的则会销毁线程，直到线程数等于核心线程数（设置allowCoreThreadTimeOut 可以回收核心线程）。

我们再回到面试题来，这个实现指的就是 Executors 的 5 个静态工厂方法：

- newFixedThreadPool
- newWorkStealingPool
- newSingleThreadExecutor
- newCachedThreadPool
- newScheduledThreadPool

###  newFixedThreadPool

![](https://cdn.jsdelivr.net/gh/yessimida/cdn_image/img/image-20210228111803637.png)

这个线程池实现特点是核心线程数和最大线程数是一致的，然后  keepAliveTime 的时间是 0 ，队列是无界队列。

按照这几个设定可以得知它任务线程数是固定，如其名 Fixed。

然后可能出现 OOM 的现象，因为队列是无界的，所以任务可能挤爆内存。

它的特性就是**我就固定出这么多线程，多余的任务就排队，就算队伍排爆了我也不管**。

因此不建议用这个方式来创建线程池。

###  newWorkStealingPool

![](https://cdn.jsdelivr.net/gh/yessimida/cdn_image/img/image-20210228111823899.png)

这个是1.8才有的，从代码可以看到返回的就是  ForkJoinPool，我们1.8用的并行流就是这个线程池。

比如`users.parallelStream().filter(...).sum();`用的就是 ForkJoinPool 。

从图中可以看到线程数会参照当前服务器可用的处理核心数，我记得并行数是核心数-1。

这个线程池的特性从名字就可以看出 Stealing，**会窃取任务**。

每个线程都有自己的**双端队列**，当自己队列的任务处理完毕之后，会去别的线程的任务队列尾部拿任务来执行，加快任务的执行速率。

至于 ForkJoin 的话，就是分而治之，把大任务分解成一个个小任务，然后分配执行之后再总和结果，再详细就自行查阅资料啦~

### newSingleThreadExecutor

![](https://cdn.jsdelivr.net/gh/yessimida/cdn_image/img/image-20210228112021332.png)

这个线程池很有个性，一个线程池就一个线程，一个人一座城，配备的也是无界队列。

它的特性就是**能保证任务是按顺序执行的**。

### newCachedThreadPool

![](https://cdn.jsdelivr.net/gh/yessimida/cdn_image/img/image-20210228112040752.png)

这个线程池是急性子，核心线程数是 0 ，最大线程数看作无限，然后**任务队列是没有存储空间的**，简单理解成来个任务就必须找个线程接着，不然就阻塞了。

cached 意思就是会缓存之前执行过的线程，缓存时间是 60 秒，这个时候如果有任务进来就可以用之前的线程来执行。

所以它适合用在**短时间内有大量短任务的场景。如果暂无可用线程，那么来个任务就会新启一个线程去执行这个任务，快速响应任务**。

但是如果任务的时间很长，那存在的线程就很多，上下文切换就很频繁，切换的消耗就很明显，并且存在太多线程在内存中，也有 OOM 的风险。

### newScheduledThreadPool

![](https://cdn.jsdelivr.net/gh/yessimida/cdn_image/img/image-20210228112108085.png)
![](https://cdn.jsdelivr.net/gh/yessimida/cdn_image/img/image-20210228112123444.png)

其实就是定时执行任务，重点就是那个延时队列。

## 什么是时间轮？

这玩意其实跟定时任务有关。

关于 Java 的几个定时任务调度相关的：Timer、DelayQueue 和 ScheduledThreadPool。这几个实现在海量定时任务前不太行，所以有个叫时间轮的实现可以顶上。

我的这篇文章分析的Netty和Kafka中的时间轮实现，算是两种不同的变型实现。

链接：[知道时间轮算法吗？在Netty和Kafka中如何应用的？](https://mp.weixin.qq.com/s/DS-Cn6yUTjQAAn5g_hk6wA)

文章也分析了Timer、DelayQueue 和 ScheduledThreadPool 的优缺点，一网打尽~

## 你都用过哪些 Java 并发工具类？

Semaphore、CyclicBarrier、CountDownLatch 三连。

当然 JUC 下面还有挺多，反正列几个说说就行，面试的时候切忌不要一股脑儿的把知道的都扔出来，这叫**留白**。

### Semaphore

这玩意叫信号量，广泛应用于各种操作系统中，相对于平日只允许一个线程访问临界区的 lock 和 synchronized 来说，**信号量允许多线程同时访问一个临界区**。

原理就简单的理解为初始化一个数，如果来了一个线程则把数减一，如果减一之后数的值小于 0 则阻塞当前线程，移入一个阻塞队列中，否则允许执行。

当一个线程执行完毕之后将数加一，并唤醒阻塞队列中的一个等待线程。

实际是内部有个继承自 AQS 的 Sync 类，通过依托 AQS 的封装来实现功能。

主要用于流量的控制，比如停车场只允许停一定数量的车位。

简单示例如下：

```
 int count;
    final Semaphore semaphore   = new Semaphore(1); // 初始化信号量
    // 用信号量保证互斥    
    void addOne() {
      try {
          semaphore.acquire();   //对应down，计数减一
          count+=1;
        } catch (InterruptedException e) {
          e.printStackTrace();
        } finally {
          semaphore.release();  //对应up，计数加一
        }
    }
```

### CyclicBarrier

从名字分析，这是一个可循环的屏障。

屏障的意思是：让一组线程都运行到同一个屏障点之后，线程会阻塞等待所有线程都达到这个屏障点，然后所有线程才得以继续执行。

![](https://cdn.jsdelivr.net/gh/yessimida/cdn_image/img/image-20210303201816902.png)

来看一下用法：

![](https://cdn.jsdelivr.net/gh/yessimida/cdn_image/img/image-20210303202009731.png)

结果如下：

![](https://cdn.jsdelivr.net/gh/yessimida/cdn_image/img/image-20210303201951098.png)

它实际上是基于 ReentrantLock 和 Condition 的封装来实现这一功能的。

**原理我先口述一下，因为面试官很有可能会问原理**。

首先设置了达到屏障的线程数量，当线程调用 await 的时候计数器会减一，如果计数器减一不等于 0 的时候，线程会调用 condition.await 进行阻塞等待。

如果计数器减一的值等于0，说明最后一个线程也到达了屏障，于是如果有 barrierCommand 就执行 barrierCommand ，然后调用 condition.signalAll 唤醒之前等待的线程，并且重置计数器，然后开启下一代。

源码我就不贴了，建议自己看下，不难的，算上一大推注释都不到 500 行，核心方法就 60 几行。

至于循环的话，来看一下这个代码就理解了：

![](https://cdn.jsdelivr.net/gh/yessimida/cdn_image/img/image-20210303202032550.png)

当规定数量的线程到达屏障之后会把计数重置回去，并且开启了下一代，所以 CyclicBarrier 是可以循环使用的。

### CountDownLatch

这个锁其实和 CyclicBarrier 有点类似，都是等待一个节点的到达，但是还是不太一样的。

CyclicBarrier 是各个线程等待阻塞所有线程都达到一个节点之后，所有线程继续执行。

CountDownLatch 是一个线程阻塞着等待其他线程到达一个节点之后才能继续执行，**这个过程中其他线程是不会阻塞的**。

示例如下：

![](https://cdn.jsdelivr.net/gh/yessimida/cdn_image/img/image-20210303202051580.png)

结果如下：

![](https://cdn.jsdelivr.net/gh/yessimida/cdn_image/img/image-20210303202159994.png)

实现原理：内部又一个继承自 AQS 的 Sync 类，核心其实就是围绕一个整数 state。

初始化 state 的值，当调用一次 countDown 会把 state 的值减一，当 state 的值减到 0 的时候就会唤醒之前调用 await 等待的线程。

主要是依靠 AQS 封装的好，所以代码很少，原理也很清晰简单。

### StampedLock

既然30题提到了这个，之前也专门写过文章分析，那刚好拿来讲讲。

可以认为是读写锁的“改进”版本。读写锁读写是互斥的，而 StampedLock 搞了个悲观读和乐观读，悲观读和写是互斥的，乐观读则不会。

搞个官方示例看下就很清晰了:

```
 class Point {
   private double x, y;
   private final StampedLock sl = new StampedLock();

   void move(double deltaX, double deltaY) { // an exclusively locked method
     long stamp = sl.writeLock();  //获取写锁
     try {
       x += deltaX;
       y += deltaY;
     } finally {
       sl.unlockWrite(stamp); //释放写锁
     }
   }

   double distanceFromOrigin() { // A read-only method
     long stamp = sl.tryOptimisticRead(); //乐观读
     double currentX = x, currentY = y;
     if (!sl.validate(stamp)) { //判断共享变量是否已经被其他线程写过
        stamp = sl.readLock();  //如果被写过则升级为悲观读锁
        try {
          currentX = x;
          currentY = y;
        } finally {
           sl.unlockRead(stamp); //释放悲观读锁
        }
     }
     return Math.sqrt(currentX * currentX + currentY * currentY);
   }

   void moveIfAtOrigin(double newX, double newY) { // upgrade
     // Could instead start with optimistic, not read mode
     long stamp = sl.readLock(); //获取读锁
     try {
       while (x == 0.0 && y == 0.0) {
         long ws = sl.tryConvertToWriteLock(stamp);  //升级为写锁
         if (ws != 0L) {
           stamp = ws;
           x = newX;
           y = newY;
           break;
         }
         else {
           sl.unlockRead(stamp);
           stamp = sl.writeLock();
         }
       }
     } finally {
       sl.unlock(stamp);
     }
   }
 }
```

乐观锁就是获取判断一下，如果被修改了那么就升级为悲观锁。

但是 StampedLock 是不可重入锁，而且也不支持 condition 。并且如果线程使用writeLock() 或者readLock() 获得锁之后，线程还没执行完就被 interrupt() 的话，会导致CPU飙升，需要用 `readLockInterruptibly 或者 writeLockInterruptibly `。

## Java 中的阻塞队列用过哪些?

阻塞队列主要用来阻塞队列的插入和获取操作，当队列满了的时候阻塞队列的插入操作，直到队列有空位。当队列为空的时候阻塞队列的获取操作，直到队列有值。

常用在实现生产者和消费者场景，在笔试题中比较常见。

常见的有 ArrayBlockingQueue 和 LinkedBlockingQueue，分别是基于数组和链表的有界阻塞队列。

两者原理都是基于 ReentrantLock 和 Condition 。

> 都是有界阻塞队列两者有什么区别？

ArrayBlockingQueue 基于数组，内部实现只用了一把锁，可以指定公平或者非公平锁。

LinkedBlockingQueue 基于链表，内部实现用了两把锁，take 一把、put 一把，所以入队和出队这两个操作是可以并行的，从这里看并发度应该比 ArrayBlockingQueue 高。

还有 PriorityBlockingQueue  和 DelayQueue，分别是支持优先级的无界阻塞队列和支持延时获取的无界阻塞队列，如果你看过 DelayQueue 实现就会发现内部用的是 PriorityQueue。

![](https://cdn.jsdelivr.net/gh/yessimida/cdn_image/img/image-20210303202224062.png)

还有 SynchronousQueue、 LinkedBlockingDeque 和 LinkedTransferQueue。

SynchronousQueue 前面线程池分析有提到过，它是不占空间的，入队比如等待一个出队，也就是生产者必须等待消费者拿货，无法把先把货存在队列。

LinkedBlockingDeque 是双端阻塞无界队列，就是队列的头尾都能操作，头尾都能插入和移除。

LinkedTransferQueue，相对于其他阻塞队列从名字来看它有 Transfer 功能，其实也不是什么神奇功能，一般阻塞队列都是将元素入队，然后消费者从队列中获取元素。

而 LinkedTransferQueue 的 transfer 是元素入队的时候看看是否已经有消费者在等了，如果有在等了直接给消费者即可，所以就是这里少了一层，没有锁操作。

## 用过Java 中哪些原子类？

原子类是 JUC 封装的通过无锁的方式实现的一系列线程安全的原子操作类。

![](https://cdn.jsdelivr.net/gh/yessimida/cdn_image/img/image-20210303202239940.png)

上面截图的原子类主要分为五大类，我画个脑图汇总一下：

![](https://cdn.jsdelivr.net/gh/yessimida/cdn_image/img/image-20210303202253776.png)

原子类的核心原理就是基于 CAS（Compare And Swap）。

CAS 简单的理解为：给予一个共享变量的内存地址，然后内存中应该的值(预期值)和新值，然后通过**一条 CPU 指令**来比较此内存地址上的值是否等于预期值，如果是则替换内存地址上的值为新值，如果不是则不予替换且换回。

也就是说硬件层面支持一条指令来实现这么几个操作，一条指令是不会被打断的，所以保证了原子性。

### 基本类型

可以简单的理解为通过基本类型原子类 AtomicBoolean、AtomicInteger 和 AtomicLong 就可线程安全地、原子地更新这几个基本类型。

### 数组类型

AtomicIntegerArray、AtomicLongArray 和 AtomicReferenceArray，简单的理解为可以原子化地更新数组内的每个元素，几个的差别无非就是数组里面存储的数据是什么类型。

### 引用类型

AtomicReference、AtomicStampedReference 和 AtomicMarkableReference，就是对象引用的原子化更新。

差别在于 AtomicStampedReference 和 AtomicMarkableReference 可以避免 CAS 的 ABA 问题。

AtomicStampedReference  是通过版本号 stamp 来避免， AtomicMarkableReference  是通过一个布尔值 mark 来避免。

> ABA 问题

因为 CAS 是将期望值和当时内存地址上的值进行对比，假设期望值是 1 ，地址上的值现在是 1，只不过中间被人改成了 2 ，然后又改回了1，所以此时你 CAS 操作去对比是可以替换的，你无法得知中间值是否改过，这种情况就叫 ABA 问题。

而解决 ABA 问题的做法就是用版本号，每次修改版本就+1，这样即使值是一样的但是版本不同，就能得知之前被改过了。

### 属性更新类型

AtomicIntegerFieldUpdater、AtomicLongFieldUpdater 和 AtomicReferenceFieldUpdater，是通过反射，原子化的更新对象的属性，不过要求属性必须用 volatile 修饰来保证可见性，看下源码，很直观。

![](https://cdn.jsdelivr.net/gh/yessimida/cdn_image/img/image-20210303202310901.png)

### 累加器

上述的都是更新数据，而 DoubleAccumulator、DoubleAdder、LongAccumulator 和 LongAdder 主要用来累加数据。

首先 AtomicLong  也能累加，而 LongAdder 是专业累加，也只能累加，并发度更高，它通过分多个 cells 来减少线程的竞争，提高了并发度。

你可以理解为如果拿 AtomicLong  是实现累加就是一本本子，然后 20 个人要让本子上累加计数。

而 LongAdder 分了 10 个本子，20个人可以分别拿这 10 个本子来计数(减少了竞争，提高了并发度)，然后最后的结果再由 10个本子上的数相加即可。

> xxxAccumulator 和 xxxAdder 两者的区别?

xxxAccumulator 的功能比 xxxAdder 丰富，可以自定义累加方法，也可以设置初始值，按照注释上的解释 xxxAdder  等价于 new xxxAccumulator((x, y) -> x + y, 0L}。

所以可以说 xxxAdder 是 xxxAccumulator 的一个特例。

## Synchronized 和 ReentrantLock 区别？

Synchronized 和 ReentrantLock 都是可重入锁，ReentrantLock 需要手动解锁，而 Synchronized 不需要。

ReentrantLock 支持设置超时时间，可以避免死锁，比较灵活，并且支持公平锁，可中断，支持条件判断。

Synchronized 不支持超时，非公平，不可中断，不支持条件。

总的而言，一般情况下用 Synchronized 足矣，比较简单，而 ReentrantLock 比较灵活，支持的功能比较多，所以复杂的情况用 ReentrantLock 。

至于说 Synchronized 性能不如 ReentrantLock 的，那都是 N 多年前的事儿了。

## Synchronized 原理知道不？

Synchronized 的原理其实就是基于一个锁对象和锁对象相关联的一个 monitor 对象。

在偏向锁和轻量级锁的时候只需要利用 CAS 来操控锁对象头即可完成加解锁动作。

在升级为重量级锁之后还需要利用 monitor 对象，利用 CAS 和 mutex 来作为底层实现。

monitor 对象颞部会有等待队列和条件等待队列，未竞争到锁的线程存储到等待队列中，获得锁的线程调用 wait 后便存放在条件等待队列中，解锁和 notify 都会唤醒相应队列中的等待线程来争抢锁。

然后由于阻塞和唤醒依赖于底层的操作系统实现，系统调用存在用户态与内核态之间的切换，所以有较高的开销，**因此称之为重量级锁**。

所以才会有偏向锁和轻量级锁的优化，并且引入自适应自旋机制，来提高锁的性能。

关于 Synchronized 其实我写过两篇文章，看完这两篇文章你可以跟面试官说，你看过 JVM 源码，毫不夸张，因为就是从源码级别上分析的。

而且指明了一个几乎网上都错了的观点和一个常见的认知错误。

总而言之，看完之后对 Synchronized 基本上超越很多人了。

## 有深入分析过Synchronized底层原理吗

我写过篇深入 JVM 源码的分析，而且都用中文注释标注了，应该看完能清晰具体的实现。

![](https://cdn.jsdelivr.net/gh/yessimida/cdn_image/img/image-20220123162247906.png)

具体答案放在下一题（Synchronized轻量级锁升级会自旋）

## Synchronized轻量级锁升级会自旋？

网上很多文章说回自旋，我看了源码，**并不会自旋**，而是直接升级成重量级锁。

链接：[Synchronized 深入JVM分析](https://mp.weixin.qq.com/s/3PBGQBR9DNphE7jSMvOHXQ)

## Synchronized 升级到重量级锁会怎样？

专门写了篇文章。

链接：[Synchronized 升级到重量级锁之后就下不来了？你错了！](https://mp.weixin.qq.com/s/SUzNj4MsRJJzkhiquuLy0Q)

## 什么是锁的自适应自旋？

这里指的就是 Syncronized 在身为重量级锁时候的自旋。

具体指的是在重量级锁时，一个线程如果竞争锁失败会进行自旋操作，说白了就是执行一些无意义的执行，空转 CPU 等着锁的释放。

因为一些情况下可能线程刚被阻塞，锁就被释放了，这样开销就比较大，所以自旋在一定程度上是有优化的。

形象一点就像怠速停车和熄火的区别，如果等待时候很长(长时候都拿不到锁)，那肯定熄火划算(阻塞)。

如果一会儿就要出发(拿到锁)，那怠速停车(自旋)比较划算。

不过因为这个自旋次数不好判断，所以**引入自适应自旋**。

说白了就是结合经验值来看，如果上次自旋一会儿就拿到锁，那这次多自旋几次，如果上次自旋很久都拿不到，这次就少自旋。

这就叫锁的自适应自旋。

## 锁如何优化？

要注意锁的粒度，不能粗暴的直接在方法外围定义锁，锁的代码块越小越好，像双检锁就是典型的优化。

不同场景定义不同的锁，不能粗暴的一把锁搞定，例如在读多写少的场景可以使用读写锁、写时复制等。

再具体的可以看看我之前写的：[Java锁事](https://mp.weixin.qq.com/s/aM87lQD9Qi6uKUa-MG0wTg)。

## ReentrantLock 的原理？

ReentrantLock 其实就是基于 AQS 实现的一个可重入锁，支持公平和非公平两种方式。

内部实现其实就是依靠一个 state 变量和两个等待队列：同步队列和等待队列。

利用 CAS 修改 state 来争抢锁。

争抢不到则入同步队列等待，同步队列是一个双向链表。

条件 condition 不满足时候则入等待队列等待，也是个双向链表。

是否是公平锁的区别在于：线程获取锁时是加入到同步队列尾部还是直接利用 CAS 争抢锁。

就是这么回事儿，理解起来应该不难，操心的我再画个图，嘿嘿。

![](https://cdn.jsdelivr.net/gh/yessimida/cdn_image/img/image-20210303213552001.png)



## 说说 AQS 吧？

AQS 的原理其实就是上面提到的，这里就不再赘述了。

如果面试官问你为什么需要 AQS ，就这样回答。

**AQS 将一些操作封装起来，比如入队等基本方法，暴露出方法**，便于其他相关 JUC 锁的使用。

比如 CountDownLatch、Semaphore 等等。

就是起到了一个抽象，封装的作用。

## 读写锁知道不?

读写锁在 Java 中一般默认指的是 ReentrantReadWriteLock。

读写锁是有两把锁，分别是读锁和写锁。

除了读读操作不互斥之外，其他都互斥。

所以读很多写比较少的情况，用读写锁比较合适。

还有一点要注意，如果不是这种情况不要用读写锁，**因为读写锁需要额外维护读锁的状态，所以如果读读操作不多还不如一般的锁**。

读写锁也是基于 AQS 实现的，再具体点的实现就是将 state分为了两部分，高16bit用于标识读状态、低16bit标识写状态。

就这样灵巧的通过一个 state 实现了两把锁，嘿嘿。

## CAS 知道不？

CAS 就是 compare and swap，即比较并交换。

举个例子，我们经常有累加需求，比较一个值是否等于 1，如果等于 1 我们将它替换成 2，如果等于 2 替换成 3。

这种比较在多线程的情况下就不安全，比如此时同时有两个线程执行到比较值是否等于 1，然后两个线程发现都等于 1。

然后两个线程都将它变成了 2，这样明明加了两次，值却等于 2。

![](https://cdn.jsdelivr.net/gh/yessimida/cdn_image/img/image-20210307091936175.png)

这种情况其实加锁可以解决，但是加锁是比较消耗资源的。

因此硬件层面就给予支持，将这个比较和交换的动作封装成一个指令，这样就保证了原子性，不会判断值确实等于 1，但是替换的时候值以及不等于 1了。

这指令就是 CAS。

CAS 需要三个操作数，分别是旧的预期值(图中的1)，变量内存地址(图中a的内存地址)，新值(图中的2)。

**指令是根据变量地址拿到值，比较是否和预期值相等，如果是的话则替换成新值，如果不是则不替换**。

其实 33 题已经提过这个了，包括 ABA 问题，之所以再写一下是可以从我提供的思路跟面试官说。

**不要一上来就三个操作数**。

你把遇到的场景(上面说的累加)，然后多线程不安全，然后用锁不好，然后硬件提供了这个指令。

按这样的思路说出来，如果我是面试官，我一听就会觉得，小伙子可以。

## 说说线程的生命周期？

可以分为初始状态、可运行状态、终止状态和休眠状态四大类。

![](https://cdn.jsdelivr.net/gh/yessimida/cdn_image/img/image-20210307100732970.png)

线程新建的时候就是初始状态，还未start。

可运行状态就是可以运行，可能正在运行，也可能正在等 CPU 时间片。

休眠状态分为三种，一种是等待锁的 blocked 状态，一种是等待条件的 waitting 状态，或者有时间限制的等待 timed_waitting 状态。

- 等待条件的操作有：Object.wait、Thread.join、LockSupport.park() 
- 时间等待就是上面设置了timeout参数的方法，例如Object.wait(1000)。

终止状态就是线程结束执行了，可以是结束任务后的自动结束，也可以是产生了异常而结束。

## 什么是 JMM ？

JMM 即 Java Memory Model，Java 内存模型。

JMM 其实是一组规则，规定了一个线程的写操作何时会对另一个线程可见(JSR133)。

**抽象的**来看 JMM 会把内存分为本地内存和主存，每个线程都有自己的私有化的本地内存，然后还有个存储共享数据的主存。

由 JMM 来定义这两个内存之间的交互规则。

这里要注意本地内存只是一种抽象的说法，实际指代：寄存器、CPU 缓存等等。

总之 JMM 屏蔽了各大底层硬件的细节，是抽象出来的一套虚拟机层面的内存规范。

## 说说原子性、可见性、有序性？

> 原子性

指的是一个操作不会被中断，要么这个操作执行完毕，要么不会执行，不会有执行一半的存在。

> 可见性

指的是一个线程对某个共享变量进行了修改，则其他线程能立刻获取到最新值。

> 有序性

指的是编译器或者处理器会将指令进行重排，这种操作会影响多线程的执行顺序导致错误。

## happens-before 听过吗？

这个问题估计应该都是来自《深入理解Java虚拟机》这本书的。

happens-before 就是定义的一些规则，在一些特定场景下，一些操作会先行发生于另一些操作。

A先行发生于B，其实含义就是 A 操作得到的结果在 B 操作开始时可以得到，重点不在于 A 执行的时间比 B 早，而是 A 的结果是可以在 B 开始时候被 B 读取的。

这是 JVM 规定的有序性，你也可以认为写 JVM 的程序员需要按照这样的规则来实现 JVM。

如操作符合以下的规则就会按照下面的定义动作先行发生。

- 程序次序规则：在一个线程内，按照程序代码顺序，书写在前面的操作先行发生于书写在后面的操作。准确地说，应该是控制流顺序而不是程序代码顺序，因为要考虑分支、循环等结构。
- 管程锁定规则：一个unlock操作先行发生于后面对同一个锁的lock操作。这里必须强调的是同一个锁，而“后面”是指时间上的先后顺序。
- volatile变量规则：对一个volatile变量的写操作先行发生于后面对这个变量的读操作，这里的“后面”同样是指时间上的先后顺序。
- 传递性规则：如果操作A先行发生于操作B，操作B先行发生于操作C，那就可以得出操作A先行发生于操作C的结论。
- 线程启动规则：Thread对象的start()方法先行发生于此线程的每一个动作。
- 线程中断规则：对线程interrupt()方法的调用先行发生于被中断线程的代码检测到中断事件的发生，可以通过Thread.interrupted()方法检测到是否有中断发生。
- 线程终止规则：线程中的所有操作都先行发生于对此线程的终止检测，我们可以通过Thread.join()方法结束、Thread.isAlive()的返回值等手段检测到线程已经终止执行。
- 对象终结规则：一个对象的初始化完成（构造函数执行结束）先行发生于它的finalize()方法的开始。

## JVM 内存区域划分

Java 虚拟机运行时数据区分为程序计数器、虚拟机栈、本地方法栈、堆、方法区。

![](https://cdn.jsdelivr.net/gh/yessimida/cdn_image/img/image-20210307101627712.png)

程序计数器、虚拟机栈、本地方法栈这 3 个区域是线程私有的，会随线程消亡而自动回收，所以不需要管理。

而堆和方法区是线程共享的，所以垃圾回收器会关注这两个地方。

堆只要存放的就是平时 new 的对象。

方法区存放的就是加载的类型信息、即时编译(JIT)后的代码等。

## 指令重排知道吗？

为了提高程序执行的效率，CPU或者编译器就将执行命令重排序。

原因是因为内存访问的速度比 CPU 运行速度慢很多，因此需要编排一下执行的顺序，防止因为访问内存的比较慢的指令而使得 CPU 闲置着。

CPU 执行有个指令流水线的概念，还有分支预测等，关于这个我之前写过一篇文章，可以看下[CPU分支预测](https://mp.weixin.qq.com/s/ZuwoGRSWf2b-FEEM9ir6pQ)

总之为了提高效率就会有指令重排的情况，导致指令乱序执行的情况发生，不过会保证结果肯定是与单线程执行结果一致的，这叫 `as-if-serial`。

不过多线程就无法保证了，在 Java 中的 volatile 关键字可以禁止修饰变量前后的指令重排。

## final 可以保证可见性吗？

**不可以**。

你可能看到一些答案说可以保证可见性，**那不是我们常说的可见性**。

一般而言我们指的可见性是一个线程修改了共享变量，另一个线程可以立马得知更改，得到最新修改后的值。

而 final 并不能保证这种情况的发生，volatile 才可以。

而有些答案提到的 final 可以保证可见性，其实指的是 final 修饰的字段在构造方法初始化完成，并且期间没有把  this 传递出去，那么当构造器执行完毕之后，其他线程就能看见 final 字段的值。

如果不用 final 修饰的话，那么有可能在构造函数里面对字段的写操作被排序到外部，这样别的线程就拿不到写操作之后的值。

来看个代码就比较清晰了。

```java
public class YesFinalTest {
   final int a; 
   int b;
   static YesFinalTest testObj;

   public void YesFinalTest () { //对字段赋值
       a = 1;
       b = 2;
   }

   public static void newTestObj () {  // 此时线程 A 调用这个方法
       testObj = new YesFinalTest ();
   }

   public static void getTestObj () {  // 此时线程 B 执行这个方法
       YesFinalTest object = obj; 
       int a = object.a; //这里读到的肯定是 1
       int b = object.b; //这里读到的可能是 2
   }
}
```

对于 final 域，编译器和处理器要遵守两个重排序规则(参考自infoq程晓明)：

    1. 在构造函数内对一个 final 域的写入，与随后把这个被构造对象的引用赋值给一个引用变量，这两个操作之间不能重排序。 初次读一个包含
    2. final 域的对象的引用，与随后初次读这个 final 域，这两个操作之间不能重排序。

所以这才是 final 的可见性，这种可见性和我们在并发中常说的可见性不是一个概念！

**所以 final 无法保证可见性**！

## 为什么需要 ThreadLocal

最近不是开放三胎政策嘛，假设你有三个孩子。

现在你带着三个孩子出去逛街，路过了玩具店，三个孩子都看中了一款变形金刚。

所以你买了一个变形金刚，打算让三个孩子轮着玩。

回到家你发现，孩子因为这个玩具吵架了，三个都争着要玩，谁也不让着谁。

这时候怎么办呢？你可以去拉架，去讲道理，说服孩子轮流玩，但这很累。

所以一个简单的办法就是出去再买两个变形金刚，这样三个孩子都有各自的变形金刚，世界就暂时得到了安宁。

映射到我们今天的主题，变形金刚就是共享变量，孩子就是程序运行的线程。有多个线程(孩子)，争抢同一个共享变量(玩具)，就会产生冲突，而程序的解决办法是加锁(父母说服，讲道理，轮流玩)，但加锁就意味着性能的消耗(父母比较累)。

所以有一种解决办法就是避免共享(让每个孩子都各自拥有一个变形金刚)，这样线程之间就不需要竞争共享变量(孩子之间就不会争抢)。

所以为什么需要 ThreadLocal？

就是为了通过本地化资源来避免共享，避免了多线程竞争导致的锁等消耗。

这里需要强调一下，不是说任何东西都能直接通过避免共享来解决，因为有些时候就必须共享。

举个例子：当利用多线程同时累加一个变量的时候，**此时就必须共享**，因为一个线程的对变量的修改需要影响要另个线程，不然累加的结果就不对了。

再举个不需要共享的例子：比如现在每个线程需要判断当前请求的用户来进行权限判断，那这个用户信息其实就不需要共享，因为每个线程只需要管自己当前执行操作的用户信息，跟别的用户不需要有交集。

好了，道理很简单，这下子想必你已经清晰了 ThreadLocal 出现的缘由了。

再来看一下 ThreadLocal 使用的小 demo。

```java
public class YesThreadLocal {

    private static final ThreadLocal<String> threadLocalName = ThreadLocal.withInitial(() -> Thread.currentThread().getName());

    public static void main(String[] args) {
        for (int i = 0; i < 5; i++) {
            new Thread(() -> {
                System.out.println("threadName: " + threadLocalName.get());
            }, "yes-thread-" + i).start();
        }
    }
}

```
输出结果如下：

![](https://cdn.jsdelivr.net/gh/yessimida/cdn_image/img/20220123164953.png)

可以看到，我在 new 线程的时候，设置了每个线程名，每个线程都操作**同一个 ThreadLocal 对象**的 get 却返回的各自的线程名，是不是很神奇？

## 应该如何设计 ThreadLocal ？

那应该怎么设计 ThreadLocal 来实现以上的操作，即本地化资源呢？

我们的目标已经明确了，就是用 ThreadLocal 变量来实现线程隔离。

从代码上看，可能最直接的实现方法就是将 ThreadLocal 看做一个 map ，然后每个线程是  key，这样每个线程去调用 `ThreadLocal.get` 的时候，将自身作为 key 去 map 找，这样就能获取各自的值了！

听起来很完美？错了！

这样 ThreadLocal 就变成共享变量了，多个线程竞争 ThreadLocal ，那就得保证 ThreadLocal 的并发安全，那就得加锁了，这样绕了一圈就又回去了！

所以这个方案不行，那应该怎么做？

答案其实上面已经讲了，**是需要在每个线程的本地都存一份值**，说白了就是每个线程需要有个变量，来存储这些需要本地化资源的值，并且值有可能有多个，所以怎么弄呢？

**在线程对象内部搞个 map，把 ThreadLocal 对象自身作为 key，把它的值作为 map 的值**。

这样每个线程可以利用同一个对象作为 key ，去各自的 map 中找到对应的值。

这不就完美了嘛！比如我现在有 3 个 ThreadLocal  对象，2 个线程。

```
ThreadLocal<String> threadLocal1 =  new ThreadLocal<>();
ThreadLocal<Integer> threadLocal2 =  new ThreadLocal<>();
ThreadLocal<Integer> threadLocal3 =  new ThreadLocal<>();
```
那此时 ThreadLocal  对象和线程的关系如下图所示：

![](https://cdn.jsdelivr.net/gh/yessimida/cdn_image/img/20220123165020.png)

这样一来就满足了本地化资源的需求，每个线程维护自己的变量，互不干扰，实现了变量的线程隔离，同时也满足存储多个本地变量的需求，完美！

JDK就是这样实现的！我们来看看源码。

## 从源码看ThreadLocal 的原理

前面我们说到 Thread 对象里面会有个 map，用来保存本地变量。

我们来看下 jdk 的 Thread 实现
```
public class Thread implements Runnable {
     // 这就是我们说的那个 map 。
    ThreadLocal.ThreadLocalMap threadLocals = null;
}
```
可以看到，确实有个 map ，不过这个 map 是 ThreadLocal 的静态内部类，记住这个变量的名字 threadLocals，下面会有用的哈。

看到这里，想必有很多小伙伴会产生一个疑问。

> 竟然这个 map 是放在 Thread 里面使用，那为什么要定义成 ThreadLocal 的静态内部类呢？

首先内部类这个东西是编译层面的概念，就像语法糖一样，经过编译器之后其实内部类会提升为外部顶级类，和平日里外部定义的类没有区别，也就是说**在 JVM 中是没有内部类这个概念的**。

一般情况下非静态内部类用在内部类，跟其他类无任何关联，专属于这个外部类使用，并且也便于调用外部类的成员变量和方法，比较方便。

而静态外部类其实就等于一个顶级类，可以独立于外部类使用，所以**更多的只是表明类结构和命名空间**。

所以说这样定义的用意就是说明 ThreadLocalMap 是和 ThreadLocal 强相关的，专用于保存线程本地变量。

现在我们来看一下 ThreadLocalMap 的定义：


![](https://cdn.jsdelivr.net/gh/yessimida/cdn_image/img/20220123165036.png)

重点我已经标出来了，首先可以看到这个 ThreadLocalMap 里面有个 Entry 数组，熟悉 HashMap 的小伙伴可能有点感觉了。

这个 Entry 继承了 WeakReference 即弱引用。这里需要注意，**不是说 Entry 自己是弱引用**，看到我标注的 Entry 构造函数的 `super(k)` 没，这个 key 才是弱引用。

所以 ThreadLocalMap 里有个 Entry 的数组，这个 Entry 的 key 就是 ThreadLocal 对象，value 就是我们需要保存的值。

> 那是如何通过 key 在数组中找到 Entry 然后得到 value 的呢 ？

这就要从上面的 `threadLocalName.get()`说起，不记得这个代码的滑上去看下示例，其实就是调用 ThreadLocal 的 get 方法。

此时就进入 `ThreadLocal#get `方法中了，这里就可以得知为什么不同的线程对同一个 ThreadLocal 对象调用 get 方法竟然能得到不同的值了。

![](https://cdn.jsdelivr.net/gh/yessimida/cdn_image/img/20220123165046.png)

这个中文注释想必很清晰了吧！`ThreadLocal#get `方法首先获取当前线程，然后得到当前线程的 ThreadLocalMap 变量即 threadLocals，然后将自己作为 key 从 ThreadLocalMap 中找到 Entry ，最终返回 Entry 里面的 value 值。

这里我们再看一下 key 是如何从 ThreadLocalMap 中找到 Entry 的，即`map.getEntry(this)`是如何实现的，其实很简单。

![](https://cdn.jsdelivr.net/gh/yessimida/cdn_image/img/20220123165100.png)

可以看到 ThreadLocalMap 虽然和 HashMap 一样，都是基于数组实现的，但是它们对于 Hash 冲突的解决方法不一样，HashMap 是通过链表(红黑树)法来解决冲突，而 ThreadLocalMap 是通过**开放寻址法来解决冲突。**

听起来好像很高级，其实道理很简单，我们来看一张图就很清晰了。

![](https://cdn.jsdelivr.net/gh/yessimida/cdn_image/img/20220123165109.png)


所以说，如果通过 key 的哈希值得到的下标无法直接命中，则会将下标 +1，即继续往后遍历数组查找 Entry ，直到找到或者返回 null。

可以看到，这种 hash 冲突的解决效率其实不高，但是一般 ThreadLocal 也不会太多，所以用这种简单的办法解决即可。

至于代码中的`expungeStaleEntry`我们等下再分析，先来看下 `ThreadLocalMap#set` 方法，看看写入的怎样实现的，来看看 hash 冲突的解决方法是否和上面说的一致。

![](https://cdn.jsdelivr.net/gh/yessimida/cdn_image/img/20220123165119.png)

可以看到 set 的逻辑也很清晰，先通过 key 的 hash 值计算出一个数组下标，然后看看这个下标是否被占用了，如果被占了看看是否就是要找的 Entry ，如果是则进行更新，如果不是则下标++，即往后遍历数组，查找下一个位置，找到空位就 new 个 Entry 然后把坑给占用了。

当然，这种数组操作一般免不了阈值的判断，如果超过阈值则需要进行扩容。

上面的清理操作和 key 为空的情况，下面再做分析，这里先略过。

至此，我们已经分析了 ThreadLocalMap 的**核心操作 get 和 set **，想必你对 ThreadLocalMap 的原理已经从源码层面清晰了！

可能有些小伙伴对 key 的哈希值的来源有点疑惑，所以我再来补充一下 `key.threadLocalHashCode`的分析。

![](https://cdn.jsdelivr.net/gh/yessimida/cdn_image/img/20220123165128.png)

可以看到`key.threadLocalHashCode`其实就是调用 nextHashCode 进行一个原子类的累加。

注意看上面都是静态变量和静态方法，所以在 ThreadLocal 对象之间是共享的，然后通过固定累加一个奇怪的数字`0x61c88647`来分配 hash 值。

这个数字当然不是乱写的，是实验证明的一个值，即通过 `0x61c88647` 累加生成的值与 2 的幂取模的结果，可以较为均匀地分布在 2 的幂长度的数组中，这样可以减少 hash 冲突。

有兴趣的小伙伴可以深入研究一下，反正我没啥兴趣。

## ThreadLocal 内存泄露之为什么要用弱引用

接下来就是要解决上面挖的坑了，即 key 的弱引用、Entry 的 key 为什么可能为 null、还有清理 Entry 的操作。

之前提到过，Entry 对 key 是弱引用，**那为什么要弱引用呢？**

我们知道，如果一个对象没有强引用，**只有弱引用的话**，这个对象是活不过一次 GC 的，所以这样的设计就是为了让当外部没有对 ThreadLocal 对象有强引用的时候，可以将 ThreadLocal 对象给清理掉。

那为什么要这样设计呢？

假设 Entry 对 key 的引用是强引用，那么来看一下这个引用链：

![](https://cdn.jsdelivr.net/gh/yessimida/cdn_image/img/20220123165136.png)


从这条引用链可以得知，如果线程一直在，那么相关的 ThreadLocal 对象肯定会一直在，因为它一直被强引用着。

看到这里，可能有人会说那线程被回收之后就好了呀。

重点来了！**线程在我们应用中，常常是以线程池的方式来使用的**，比如 Tomcat 的线程池处理了一堆请求，而线程池中的线程一般是不会被清理掉的，**所以这个引用链就会一直在**，那么 ThreadLocal 对象即使没有用了，也会随着线程的存在，而一直存在着！

所以这条引用链需要弱化一下，而能操作的只有 Entry 和 key 之间的引用，所以它们之间用弱引用来实现。

与之对应的还有一个条引用链，我结合着上面的线程引用链都画出来：

![](https://cdn.jsdelivr.net/gh/yessimida/cdn_image/img/20220123165146.png)


另一条引用链就是栈上的 ThreadLocal 引用指向堆中的 ThreadLocal 对象，这个引用是强引用。

**如果有这条强引用存在，那说明此时的 ThreadLocal 是有用的**，此时如果发生 GC 则 ThreadLocal 对象不会被清除，因为有个强引用存在。

当随着方法的执行完毕，相应的栈帧也出栈了，此时这条强引用链就没了，如果没有别的栈有对 ThreadLocal 对象的引用，那么说明 ThreadLocal 对象无法再被访问到(定义成静态变量的另说)。

那此时 ThreadLocal 只存在与 Entry 之间的弱引用，那此时发生 GC 它就可以被清除了，因为它无法被外部使用了，那就等于没用了，是个垃圾，应该被处理来节省空间。

至此，想必你已经明白为什么 Entry 和 key 之间要设计为弱引用，就是因为平日线程的使用方式基本上都是线程池，所以线程的生命周期就很长，可能从你部署上线后一直存在，而 ThreadLocal 对象的生命周期可能没这么长。

所以为了能让已经没用 ThreadLocal 对象得以回收，所以 Entry 和 key 要设计成弱引用，不然 Entry 和 key是强引用的话，ThreadLocal 对象就会一直在内存中存在。

但是这样设计就可能产生内存泄漏。

那什么叫内存泄漏？就是指：程序中已经无用的内存无法被释放，造成系统内存的浪费。

当 Entry 中的 key 即 ThreadLocal 对象被回收了之后，**会发生 Entry 中 key 为 null 的情况**，其实这个 Entry 就已经没用了，但是又无法被回收，因为有 Thread->ThreadLocalMap ->Entry 这条强引用在，这样没用的内存无法被回收就是内存泄露。

那既然会有内存泄漏还这样实现？

这里就要填一填上面的坑了，也就是涉及到的关于 `expungeStaleEntry`即清理过期的 Entry 的操作。

设计者当然知道会出现这种情况，所以在多个地方都做了清理无用 Entry ，即 key 已经被回收的 Entry 的操作。

比如通过 key 查找 Entry 的时候，如果下标无法直接命中，那么就会向后遍历数组，此时遇到 key 为 null 的 Entry 就会清理掉，再贴一下这个方法：

![](https://cdn.jsdelivr.net/gh/yessimida/cdn_image/img/20220123165155.png)

这个方法也很简单，我们来看一下它的实现：

![](https://cdn.jsdelivr.net/gh/yessimida/cdn_image/img/20220123165206.png)

所以在查找 Entry 的时候，就会顺道清理无用的 Entry ，这样就能防止一部分的内存泄露啦！

还有像扩容的时候也会清理无用的 Entry：

![](https://cdn.jsdelivr.net/gh/yessimida/cdn_image/img/20220123165219.png)

其它还有，我就不贴了，反正知晓设计者是做了一些操作来回收无用的 Entry 的即可。

## ThreadLocal 的最佳实践

当然，等着这些操作被动回收不是最好的方法，假设后面没人调用 get 或者调用 get 都直接命中或者不会发生扩容，那无用的 Entry 岂不是一直存在了吗？所以上面说只能防止一部分的内存泄露。

所以，最佳实践是用完了之后，调用一下 remove 方法，手工把 Entry 清理掉，这样就不会发生内存泄漏了！

```
void yesDosth {
	threadlocal.set(xxx);
	try {
		// do sth
	} finally {
		threadlocal.remove();
	}
}
```
这就是使用 Threadlocal 的一个正确姿势啦，即不需要的时候，显示的 remove 掉。

当然，如果不是线程池使用方式的话，其实不用关系内存泄漏，反正线程执行完了就都回收了，**但是一般我们都是使用线程池的**，可能只是你没感觉到。

比如你用了 tomcat ，其实请求的执行用的就是 tomcat 的线程池，这就是隐式使用。

还有一个问题，关于 withInitial 也就是初始化值的方法。

由于类似 tomcat 这种隐式线程池的存在，即线程第一次调用执行 Threadlocal 之后，如果没有显示调用 remove 方法，则这个 Entry 还是存在的，那么下次这个线程再执行任务的时候，不会再调用 withInitial 方法，**也就是说会拿到上一次执行的值**。

**但是你以为执行任务的是新线程，会初始化值，然而它是线程池里面的老线程，这就和预期不一致了，所以这里需要注意。**

## InheritableThreadLocal？

这玩意可以理解为就是可以把父线程的 threadlocal 传递给子线程，所以如果要这样传递就用 InheritableThreadLocal ，不要用 threadlocal。

原理其实很简单，在 Thread 中已经包含了这个成员：

![](https://cdn.jsdelivr.net/gh/yessimida/cdn_image/img/20220123165231.png)

在父线程创建子线程的时候，子线程的构造函数可以得到父线程，然后判断下父线程的 InheritableThreadLocal 是否有值，如果有的话就拷过来。

![](https://cdn.jsdelivr.net/gh/yessimida/cdn_image/img/20220123165241.png)

这里要注意，只会在线程创建的时会拷贝 InheritableThreadLocal 的值，之后父线程如何更改，子线程都不会受其影响。

## ThreadLocal 的缺点？
ThreadLocal 的一个缺点：hash 冲突用的是线性探测法，效率低。

![](https://cdn.jsdelivr.net/gh/yessimida/cdn_image/img/20220123165307.png)

可以看到，图上显示的是经过两个遍历找到了空位，假设冲突多了，需要遍历的次数就多了。并且下次 get 的时候，hash 直接命中的位置发现不是要找的 Entry ，于是就接着遍历向后找，所以说这个效率低。

而像 HashMap 是通过链表法来解决冲突，并且为了防止链表过长遍历的开销变大，在一定条件之后又会转变成红黑树来查找，这样的解决方案在频繁冲突的条件下，肯定是优于线性探测法，所以这是一个优化方向。

不过 FastThreadLocal  不是这样优化的，我们下面再说。

还有一个缺点是 ThreadLocal 使用了 WeakReference 以保证资源可以被释放，但是这可能会产生一些 Etnry 的 key 为 null，即无用的 Entry 存在。

所以调用 ThreadLocal 的 get 或 set 方法时，会主动清理无用的 Entry，减轻内存泄漏的发生。

![](https://cdn.jsdelivr.net/gh/yessimida/cdn_image/img/20220123165316.png)

这其实等于把清理的开销弄到了 get 和 set 上，万一 get 的时候清理的无用 Entry 特别多，那这次 get 相对而言就比较慢了。

还有一个就是内存泄漏的问题了，当然这个问题只存在于用线程池使用的时候，并且上面也提到了 get 和 set 的时候也能清理一些无用的 Key，所以没有那么的夸张，只要记得用完后调用 ThreadLocal#remove 就不会有内存泄漏的问题了。

大致就这么几点。

## 应该如何针对 ThreadLocal 缺点改进

前面提到 ThreadLocal hash 冲突的线性探测法不好，还有 Entry 的弱引用可能会发生内存泄漏，这些都和 ThreadLocalMap 有关，所以需要搞个新的 map 来替换 ThreadLocalMap。

而这个 ThreadLocalMap 又是 Thread 里面的一个成员变量，这么一看 Thread 也得动一动，但是我们又无法修改 Thread 的代码，所以配套的还得弄个新的 Thread。

所以我们不仅得弄个新的 ThreadLocal、ThreadLocalMap 还得弄个配套的 Thread 来用上新的 ThreadLocalMap 。

所以如果想改进 ThreadLocal ，就需要动这三个类。

对应到 Netty 的实现就是 FastThreadLocal、InternalThreadLocalMap、FastThreadLocalThread

![](https://cdn.jsdelivr.net/gh/yessimida/cdn_image/img/20220123165328.png)

然后发散一下思维，既然 Hash 冲突的想线性探测效果不好，你可能比较容易想到的就是上面提到的链表法，然后再基于链表法说个改成红黑树，这个确实是一方面，但是可以再想想。

比如，让 Hash 不冲突，所以设计一个不会冲突的 hash 算法？不存在的！

所以怎么样才不会产生冲突呢？

**各自取号入座**

什么意思？就是每往 InternalThreadLocalMap 中塞入一个新的 FastThreadLocal 对象，就给这个对象发个唯一的下标，然后让这个对象记住这个下标，到时候去 InternalThreadLocalMap 找 value 的时候，直接通过下标去取对应的 value 。

这样不就不会冲突了？

这就是 FastThreadLocal 给出的方案，具体下面分析。

还有个内存泄漏的问题，这个其实只要规范的使用即用完后 remove 就好了，其实也没太好的解决方案，不过 FastThreadLocal 曲线救国了一下，这个也且看下面的分析！

## 噢？那 FastThreadLocal 的原理是？

> 以下 Netty 基于 4.1 版本分析

先来看下 FastThreadLocal 的定义：

![](https://cdn.jsdelivr.net/gh/yessimida/cdn_image/img/20220123165337.png)

可以看到有个叫 variablesToRemoveIndex 的**类成员**，并且用 final 修饰的，所以等于每个 FastThreadLocal 都有个共同的不可变 int 值，值为多少等下分析。

然后看到这个 index 没，在 FastThreadLocal 构造的时候就被赋值了，且也被 final 修饰，所以也不可变，**这个 index 就是我上面说的给每个新 FastThreadLocal 都发个唯一的下标**，这样每个 index 就都知道自己的位置了。

上面两个 index 都是通过 `InternalThreadLocalMap.nextVariableIndex()` 赋值的，盲猜一下，这个肯定是用原子类递增实现的。

我们来看一下实现：

![](https://cdn.jsdelivr.net/gh/yessimida/cdn_image/img/20220123165346.png)

，在 InternalThreadLocalMap 也定义了一个静态原子类，每次调用 nextVariableIndex 就返回且递增，没有什么别的赋值操作，从这里也可以得知 variablesToRemoveIndex 的值为 0，因为它属于常量赋值，第一次调用时 nextIndex 的值为 0 。

 看到这，不知道大家是否已经感觉到一丝不对劲了。好像有点浪费空间的意思，我们继续往下看。

InternalThreadLocalMap 对标的就是之前的 ThreadLocalMap 也就是 ThreadLocal 缺点集中的类，需要重点看下。

我们再来回顾一下 ThreadLocalMap 的定义。 

![](https://cdn.jsdelivr.net/gh/yessimida/cdn_image/img/20220123165356.png)

它是个 Entry 数组，然后 Entry 里面弱引用了 ThreadLocal 作为 Key。

而 InternalThreadLocalMap 有点不太一样：

![](https://cdn.jsdelivr.net/gh/yessimida/cdn_image/img/20220123165405.png)

可以看到， InternalThreadLocalMap 好像放弃了 map 的形式，没用定义 key 和 value，而是一个 Object 数组？

那它是如何通过 Object 来存储  FastThreadLocal 和对应的 value 的呢？我们从 FastThreadLocal#set 开始分析：

![](https://cdn.jsdelivr.net/gh/yessimida/cdn_image/img/20220123165421.png)

 因为我们已经熟悉 ThreadLocal 的套路，所以我们知道 InternalThreadLocalMap 肯定是 FastThreadLocalThread 里面的一个变量。

然后我们从对应的 FastThreadLocalThread 里面拿到了 map 之后，就要执行塞入操作即 `setKnownNotUnset`。

我们先看一下塞入操作里面的 `setIndexedVariable` 方法：

![](https://img-blog.csdnimg.cn/a69c35fde9764dda9154b8ebd3ec288b.png)

可以看到，根据传入构造 FastThreadLocal 生成的唯一 index 可以直接从 Object 数组里面找到下标并且进行替换，这样一来压根就不会产生冲突，逻辑很简单，完美。

那如果塞入的 value 不是 UNSET(默认值)，则执行 `addToVariablesToRemove` 方法，这个方法又有什么用呢？

![](https://cdn.jsdelivr.net/gh/yessimida/cdn_image/img/20220123165431.png)

是不是看着有点奇怪？这是啥操作？别急，看我画个图来解释解释：

![](https://cdn.jsdelivr.net/gh/yessimida/cdn_image/img/20220123165443.png)

这就是 Object 数组的核心关系图了，第一个位置放了一个 set ，set 里面存储了所有使用的 FastThreadLocal 对象，然后数组后面的位置都放 value。

那为什么要放一个 set 保存所有使用的 FastThreadLocal 对象？

**用于删除**，你想想看，假设现在要清空线程里面的所有 FastThreadLocal ，那必然得有一个地方来存放这些 FastThreadLocal 对象，这样才能找到这些家伙，然后干掉。

所以刚好就把数组的第一个位置腾出来放一个 set 来保存这些 FastThreadLocal 对象，如果要删除全部 FastThreadLocal 对象的时候，只需要遍历这个 set ，得到 FastThreadLocal 的 index 找到数组对应的 位置将 value 置空，然后把 FastThreadLocal 从 set 中移除即可。

刚好 FastThreadLocal 里面实现了这个方法，我们来看下：

![](https://cdn.jsdelivr.net/gh/yessimida/cdn_image/img/20220123165452.png)

内容可能有点多了，我们做下小结，理一理上面说的：

首先 InternalThreadLocalMap 没有采用 ThreadLocalMap k-v形式的存储方式，而是用 Object 数组来存储 FastThreadLocal 对象和其 value，具体是在第一个位置存放了一个包含所使用的 FastThreadLocal 对象的 set，然后后面存储所有的 value。

之所以需要个 set 是为了存储所有使用的 FastThreadLocal 对象，这样就能找到这些对象，便于后面的删除工作。

之所以数组其他位置可以直接存储 value ，是因为每个 FastThreadLocal 构造的时候已经被分配了一个唯一的下标，这个下标对应的就是 value 所处的下标。

看到这里，不知道大家是否有感受到空间的浪费？

我举个例子。

假设系统里面一个 new 了 100 个 FastThreadLocal ，那第 100 个 FastThreadLocal 的下标就是 100 ，这个应该没有疑义。

从上面的 set 方法可以得知，只有调用 set 的时候，才会从当前线程中拿出 InternalThreadLocalMap ，然后往这个 map 的数组里面塞入 value，这里我们再回顾一下 set 的方法。

![](https://cdn.jsdelivr.net/gh/yessimida/cdn_image/img/20220123165504.png)

那这里是什么意思呢？

如果我这个线程之前都没塞过 FastThreadLocal ，此时要塞入**第一个** FastThreadLocal ，构造出来的数组长度是32，但是这个 FastThreadLocal 的下标已经涨到了 100 了，所以**这个线程第一次塞值，也仅仅只有这么一个值，数组就需要扩容**。

看到没，这就是我所说的浪费，空间被浪费了。

Netty 相关实现者知道这样会浪费空间，**所以数组的扩容是基于 index 而不是原先数组的大小**，你看看如果是基于原先数组的扩容，那么第一次扩容 2 倍，32 变成 64，还是塞不下下标 100 的数据，所以还得扩容一次，这就不美了。

所以可以看到扩容传进去的参数是 index 。

![](https://cdn.jsdelivr.net/gh/yessimida/cdn_image/img/20220123165514.png)

可以看到，直接基于 index 的向上 2 次幂取整。然后就是扩容的拷贝，这里是直接进行数组拷贝，**不需要进行 rehash**，而 ThreadLocalMap 的扩容需要进行rehash，也就是重新基于 key 的 hash 值进行位置的分配，所以**这个也是 FastThreadLocal 优于ThreadLocal 的一个点**。

对了，上面那个向上 2 次幂取整的操作，不知道你们熟悉不熟悉，这个和 HashMap 的实现是一致的。

![](https://cdn.jsdelivr.net/gh/yessimida/cdn_image/img/20220123165523.png)

咳咳，但是我没有证据，只能说优秀的代码，就是**源远流长**。

所以从上面的实现可以得知 Netty 就是特意这样设计的，用多余的空间去换取不会冲突的 set 和 get ，这样写入和获取的速度就更快了，这就是典型的**空间换时间**。

好了，想必此时你已经弄懂了 FastThreadLocal 的核心原理了，我们再来看看 get 方法的实现，我想你应该能脑补这个实现了。

![](https://cdn.jsdelivr.net/gh/yessimida/cdn_image/img/20220123165541.png)

是吧，没啥难度，index 就是 FastThreadLocal 构造时候预先分配好的那个下标，然后直接进行一个数组下标查找，如果没找到就调用 init 方法进行初始化。

我们这里再继续探究一下`InternalThreadLocalMap.get()`，这里面做了一个兼容。不过我要先介绍一下 FastThreadLocalThread ，就是这玩意替代了  Thread。

![](https://cdn.jsdelivr.net/gh/yessimida/cdn_image/img/20220123165552.png)

可以看到它继承了 Thread ，并且弄了一个成员变量就是我们前面说的 InternalThreadLocalMap。

然后我们再来看一下 get 方法，我截了好几个，不过逻辑很简单。

![](https://cdn.jsdelivr.net/gh/yessimida/cdn_image/img/20220123165601.png)

这里之所以分了 fastGet 和 slowGet 是为了做一个兼容，假设有个不熟悉的人，他用了 FastThreadLocal 但是没有配套使用 FastThreadLocalThread ，然后调用 FastThreadLocal#get 的时候去 Thread 里面找 InternalThreadLocalMap 那不就傻了吗，会报错的。

所以就再弄了个 slowThreadLocalMap ，它是个 ThreadLocal ，里面保存 InternalThreadLocalMap 来兼容一下这个情况。

从这里我们也能得知，FastThreadLocal 最好和 FastThreadLocalThread 配套使用，不然就隔了一层了。

```
FastThreadLocal<String> threadLocal = new FastThreadLocal<String>();
Thread t = new FastThreadLocalThread(new Runnable() { //记得要 new FastThreadLocalThread
     public void run() {
	     threadLocal.get()；
	     ....
     }
 });
```

好了，get 和 set 这两个核心操作都分析完了，我们最后再来看一下 remove 操作吧。

![](https://cdn.jsdelivr.net/gh/yessimida/cdn_image/img/20220123165612.png)

很简单对吧，把数组里的 value 给覆盖了，然后再到  set  里把对应的 FastThreadLocal 对象给删了。

不过看到这里，可能有人会发出疑惑，内存泄漏相关的点呢？

其实吧，可以看到 FastThreadLocal 就没用弱引用，所以它把无用 FastThreadLocal 的清理就寄托到规范使用上，即没用了就主动调用 remove 方法。

但是它曲线救国了一下，我们来看一下 FastThreadLocalRunnable 这个类：

![](https://cdn.jsdelivr.net/gh/yessimida/cdn_image/img/20220123165624.png)

我已经把重点画出来了，可以看到这个 Runnable 执行完毕之后，会主动调用 FastThreadLocal.removeAll() 来清理所有的 FastThreadLocal，这就是我说的曲线救国，怕你完了调用 remove ，没事我帮你封装一下，就是这么贴心。

当然，这个前提是你不能用 Runnable 而是用 FastThreadLocalRunnable。不过这里 Netty 也是做了封装的。

Netty 实现了一个 DefaultThreadFactory 工厂类来创建线程。

![](https://cdn.jsdelivr.net/gh/yessimida/cdn_image/img/20220123165632.png)

你看，你传入 Runnable 是吧，没事，我把它包成 FastThreadLocalRunnable，并且我 new 回去的线程是 FastThreadLocalThread 类型，这样就能在很大程度上避免使用的错误，也减少了使用的难度。

这也是工厂方法这个设计模式的好处之一啦。所以工程上如果怕对方没用对，我们就封装了再给别人使用，这样也屏蔽了一些细节，**他好你也好**。

所以说多看看开源框架的源码，有很多可以学习的地方！好了，FastThreadLocal 原理大致就说到这里。

## TransmittableThreadLocal 听过吗？

这玩意是阿里开源的一个组件，原生的 ThreadLocal 不支持在线程池中传递本地变量，所以弄了个这个来实现下这个需求

![](https://cdn.jsdelivr.net/gh/yessimida/cdn_image/img/image-20220123165841583.png)

详情看我写的这篇文章即可：

[TransmittableThreadLocal原理分析](https://mp.weixin.qq.com/s/JTW82eTZRyKbkgOe2E_qEg)

---

TBC。

有更多相关的Java并发面试题，可以提PR哈，有错误欢迎联系我。

除了这个系列，我的公众号每周至少都会有一篇原创，欢迎关注~

![](https://gitee.com/yessimida/interview-of-legends/raw/master/pic/16034279-e6ebb79b5a0b8fe7.png)

最近已经汇集了近 500 名朋友交流各大小厂面试真题，也期待各位的面试题分享，公众号有我的联系方式，如果有兴趣可以加我备注 **面霸**，拉你进真题交流群。

![](https://cdn.jsdelivr.net/gh/yessimida/cdn_image/img/image-20210228190741512.png)
