---
title: java-lock
tags:
  - java
categories:
  - 技术
date: 2022-12-02 13:03:48
---

# Java锁

# 对象头

Java对象保存在内存中时，由以下三部分组成：对象头、实例数据、对齐填充字节。

java的对象头由以下三部分组成：Mark Word、指向类的指针、数组长度（只有数组对象才有）

Mark Word在不同的锁状态下存储的内容不同，在32位JVM中是这么存的：

![img](https://vipkshttps8.wiz.cn/editor/63531d60-bd31-11ea-aa67-1778e2b2bbbd/20aa8caa-8757-4902-b032-92d733f8b349/resources/GbgLyBOmx3o2FIOTQSUJ1omMwf2bJqaQJWOIEkVNapo.png?token=W.hfyxpaKeZ1ALEo-GLY7TuIA8SX33YHKhnov6_ndm3LeF2E0ocVrbQuP2ZExmv08)

# Syschronized底层的原理

## **Monitor**

**Monitor**被翻译为监视器或管程

每个Java对象都可以关联一个Monitor对象,如果使用synchronized给对象上锁(重量级)之后【之前可能要先经过轻量级锁或者偏向锁】，该对象头的Mark Word中就被设置指向Monitor对象的指针。

Monitor的结果如下：

![img](https://vipkshttps8.wiz.cn/editor/63531d60-bd31-11ea-aa67-1778e2b2bbbd/20aa8caa-8757-4902-b032-92d733f8b349/resources/2OyH0fg-gdfN2ntPtGg3NpsMv_HHgPx-RJL7_mb--Zk.png?token=W.hfyxpaKeZ1ALEo-GLY7TuIA8SX33YHKhnov6_ndm3LeF2E0ocVrbQuP2ZExmv08)

1. 刚开始Monitor中Owner为null

1. 当Thread-2执行synchronized(obj)就会将Monitor的所有者Owner置为Thread-2, Monitor中只能有一一个Owner

1. 在Thread-2持有锁的过程中，如果Thread-3, Thread-4， Thread-5 也来执行synchronized(obj)， 就会进入EntryList BLOCKED

1. Thread-2执行完同步代码块的内容，然后唤醒EntryL ist中等待的线程来竞争锁，竞争的时是非公平的

1. 图中WaitSet中的Thread-0，Thread-1 是之前获得过锁，但条件不满足进入WAITING状态的线程，后面讲wait-notify时会分析

注意:

- synchronized必须是进入同一个对象的monitor才有上述的效果

- 不加synchronized的对象不会关联监视器， 不遵从以上规则

### 字节码层面的上理解

static final Object lock = new Object();

static int counter = 0;

public static void main(String[] args) {

   synchronized (lock) {

​     counter++;

   }

}

 Code:

​	 stack=2, locals=3, args_size=1

​	 0: getstatic #2 // 获取静态变量

​	 3: dup     // 压入操作数栈顶

​	 4: astore_1 // lock引用 存入 slot 1

​	 5: monitorenter // 将 lock对象 MarkWord 置为 Monitor 指针

​	 6: getstatic #3 // 获取counter的值

​	 9: iconst_1 // 准备常数 1

​	 10: iadd // +1

​	 11: putstatic #3 // -> i

​	 14: aload_1 // <- lock引用

​	 15: monitorexit // 将 lock对象 MarkWord 重置, 唤醒 EntryList

​	 16: goto 24

​	 19: astore_2 // e -> slot 2 

​	 20: aload_1 // <- lock引用

​	 21: monitorexit // 将 lock对象 MarkWord 重置, 唤醒 EntryList

​	 22: aload_2 // <- slot 2 (e)

​	 23: athrow // throw e

​	 24: return

Exception table:

​	 from to target type

​	 6 16 19 any

​	 19 22 19 any

 LineNumberTable:

​	 line 8: 0

​	 line 9: 6

​	 line 10: 14

​	 line 11: 24

 LocalVariableTable:

​	 Start Length Slot Name Signature

​	 0 25 0 args [Ljava/lang/String;

 StackMapTable: number_of_entries = 2

​	 frame_type = 255 /* full_frame */

​		 offset_delta = 19

​		 locals = [ class "[Ljava/lang/String;", class java/lang/Object ]

​		 stack = [ class java/lang/Throwable ]

​	 frame_type = 250 /* chop */

​		 offset_delta = 4

上面讲述的都是重量级锁的状态，其实，一开始的时候并不是重量级锁的状态，而是在竞争后升级成的，下面我们来看这个升级的过程。

# 锁升级

## 轻量级锁

轻量级锁的使用**场景**：如果一个对象虽然有多线程要加锁，但加锁的时间是错开的（也就是没有竞争），那么可以使用轻量级锁来优化。

**上锁过程**

1. 当线程Thread0执行到monitorenter时，会在线程的栈帧中创建一个名为**锁记录**的结构：

![img](https://vipkshttps8.wiz.cn/editor/63531d60-bd31-11ea-aa67-1778e2b2bbbd/20aa8caa-8757-4902-b032-92d733f8b349/resources/0nteySAHqxYPbFaPMyaIhj5qLHi5Ub9uKGZ3UzjDMCo.png?token=W.hfyxpaKeZ1ALEo-GLY7TuIA8SX33YHKhnov6_ndm3LeF2E0ocVrbQuP2ZExmv08)

1. 让锁记录中 Object reference 指向锁对象，并尝试用 cas 替换 Object【锁对象】 的 Mark Word，并将Mark Word 的值存入锁记录【解锁的时候需要替换回来】。

![img](https://vipkshttps8.wiz.cn/editor/63531d60-bd31-11ea-aa67-1778e2b2bbbd/20aa8caa-8757-4902-b032-92d733f8b349/resources/HDu2_5lSam8K4xr3kQXV0TLeY2iuK63W8ReEqC8erYE.png?token=W.hfyxpaKeZ1ALEo-GLY7TuIA8SX33YHKhnov6_ndm3LeF2E0ocVrbQuP2ZExmv08)

1. 如果 cas 替换成功，对象头中存储了 锁记录地址和状态 00 ，表示由该线程给对象加锁，这时图示如下：

![img](https://vipkshttps8.wiz.cn/editor/63531d60-bd31-11ea-aa67-1778e2b2bbbd/20aa8caa-8757-4902-b032-92d733f8b349/resources/K8DOty-AavAZSjKuaYrIwhPbGD5hluKAJEE6wnow5gI.png?token=W.hfyxpaKeZ1ALEo-GLY7TuIA8SX33YHKhnov6_ndm3LeF2E0ocVrbQuP2ZExmv08)

1. 如果 cas 失败，有两种情况 

1. 如果是其它线程已经持有了该 Object 的轻量级锁，这时表明有竞争，进入锁膨胀过程 【参考下面的章节】

1. 如果是自己执行了 synchronized 锁重入，那么再添加一条 Lock Record 作为重入的计数：

![img](https://vipkshttps8.wiz.cn/editor/63531d60-bd31-11ea-aa67-1778e2b2bbbd/20aa8caa-8757-4902-b032-92d733f8b349/resources/howonVgK01pPzobs376tI9NFTvcOHSZuvrjwhQtGCuU.png?token=W.hfyxpaKeZ1ALEo-GLY7TuIA8SX33YHKhnov6_ndm3LeF2E0ocVrbQuP2ZExmv08)

**解锁过程：**

1. 当退出 synchronized 代码块（解锁时）如果有取值为 null 的锁记录，表示有重入，这时重置锁记录，表示重入计数减一。结束

1. 当退出 synchronized 代码块（解锁时）锁记录的值不为 null，这时使用 cas 将 Mark Word 的值恢复给对象头 

1. 成功，则解锁成功 

1. 失败，说明轻量级锁进行了锁膨胀或已经升级为重量级锁，进入重量级锁解锁流程【参考monitor】

**锁膨胀过程：**

1. 当 Thread-1 进行轻量级加锁时，Thread-0 已经对该对象加了轻量级锁，这时 Thread-1 加轻量级锁失败，进入锁膨胀流程。

1. 为 Object 对象申请 Monitor 锁，让 Object markword 指向重量级锁地址,然后自己进入 Monitor 的 EntryList BLOCKED

![img](https://vipkshttps8.wiz.cn/editor/63531d60-bd31-11ea-aa67-1778e2b2bbbd/20aa8caa-8757-4902-b032-92d733f8b349/resources/pSZykt9xIoqmY0751JKmBpl3X4uxBfbi-jtFLHOnGuY.png?token=W.hfyxpaKeZ1ALEo-GLY7TuIA8SX33YHKhnov6_ndm3LeF2E0ocVrbQuP2ZExmv08)

1. 当 Thread-0 退出同步块解锁时，使用 cas 将 Mark Word 的值恢复给对象头，失败。这时会进入重量级解锁 流程，即按照 Monitor 地址找到 Monitor 对象，设置 Owner 为 null，唤醒 EntryList 中 BLOCKED 线程

## 自旋锁

重量级锁竞争的时候，还可以使用自旋来进行优化，如果当前线程自旋成功（即这时候持锁线程已经退出了同步 块，释放了锁），这时当前线程就可以避免阻塞。

这是一种侥幸心里，在进入阻塞队里之前，碰碰运气一样 的重新获取锁几次，如果成功了就不必进行上下问的切换了。

自旋会占用 CPU 时间，单核 CPU 自旋就是浪费，多核 CPU 自旋才能发挥优势。 

在 Java 6 之后自旋锁是自适应的，比如对象刚刚的一次自旋操作成功过，那么认为这次自旋成功的可能性会高，就多自旋几次；反之，就少自旋甚至不自旋，总之，比较智能。 

Java 7 之后不能控制是否开启自旋功能。

## 偏向锁

轻量级锁在没有竞争时（就自己这个线程），每次重入仍然需要执行 CAS 操作。 Java 6 中引入了偏向锁来做进一步优化：只有第一次使用 CAS 将线程 ID 设置到对象的 Mark Word 头，之后发现 这个线程 ID 是自己的，就表示没有竞争，不用重新 CAS。以后只要不发生竞争，这个对象就归该线程所有。

**加锁过程：**

1. 一个对象刚开始实例化的时候，没有任何线程来访问它的时候。它是可偏向的，意味着，它现在认为只可能有一个线程来访问它，所以当第一个线程来访问它的时候，它会偏向这个线程。

1. 此时，对象持有偏向锁。偏向第一个线程，这个线程在修改对象头成为偏向锁的时候使用CAS操作，并将对象头中的ThreadID改成自己的ID，之后再次访问这个对象时，只需要对比ID，不需要再使用CAS在进行操作。

1. 一旦有第二个线程访问这个对象，因为偏向锁不会主动释放，所以第二个线程可以看到对象的偏向状态，这时表明在这个对象上已经存在竞争了。等待偏向线程到达安全点，进行暂停，检查原来持有该对象锁的线程是否依然存活，如果挂了，则可以将对象变为无锁状态，然后重新偏向新的线程

1. 如果原来的线程依然存活，那么就会将偏向锁升级为轻量级锁，然后唤醒线程 A 执行完后续操作，线程 B 自旋获取轻量级锁。

**锁撤销过程：**

- 调用了对象的 hashCode，但偏向锁的对象 MarkWord 中存储的是线程 id，如果调用 hashCode 会导致偏向锁被 撤销 。

- 轻量级锁会在锁记录中记录 hashCode 

- 重量级锁会在 Monitor 中记录 hashCode

**小知识：**

- 如果开启了偏向锁（默认开启），那么对象创建后，markword 值为 0x05 即最后 3 位为 101，这时它的 thread、epoch、age 都为 0 

- 偏向锁是默认是延迟的，不会在程序启动时立即生效，如果想避免延迟，可以加 VM 参数 - XX:BiasedLockingStartupDelay=0 来禁用延迟 

- 如果没有开启偏向锁，那么对象创建后，markword 值为 0x01 即最后 3 位为 001，这时它的 hashcode、 age 都为 0，第一次用到 hashcode 时才会赋值

# 锁消除

锁消除是指虚拟机即时编译器在运行时，对一些代码上要求同步，但是被检测到不可能存在共享数据竞争的锁进行消除。锁消除的主要判定依据来源于逃逸分析的数据支持（第11章已经讲解过逃逸分析技术），如果判断在一段代码中，堆上的所有数据都不会逃逸出去从而被其他线程访问到，那就可以把它们当做栈上数据对待，认为它们是线程私有的，同步加锁自然就无须进行。

![img](https://vipkshttps8.wiz.cn/editor/63531d60-bd31-11ea-aa67-1778e2b2bbbd/20aa8caa-8757-4902-b032-92d733f8b349/resources/37SsuK_b9OaRKKLr7CVZyaL1UbdZEhZPMQmRy1pWbd4.png?token=W.hfyxpaKeZ1ALEo-GLY7TuIA8SX33YHKhnov6_ndm3LeF2E0ocVrbQuP2ZExmv08)

每个StringBuffer.append()方法中都有一个同步块，锁就是sb对象。虚拟机观察变量sb，很快就会发现它的动态作用域被限制在concatString()方法内部。也就是说，sb的所有引用永远不会“逃逸”到concatString()方法之外，其他线程无法访问到它，因此，虽然这里有锁，但是可以被安全地消除掉，在即时编译之后，这段代码就会忽略掉所有的同步而直接执行了。

# 锁粗化

原则上，我们在编写代码的时候，总是推荐将同步块的作用范围限制得尽量小——只在共享数据的实际作用域中才进行同步，这样是为了使得需要同步的操作数量尽可能变小，如果存在锁竞争，那等待锁的线程也能尽快拿到锁。大部分情况下，上面的原则都是正确的，但是如果一系列的连续操作都对同一个对象反复加锁和解锁，甚至加锁操作是出现在循环体中的，那即使没有线程竞争，频繁地进行互斥同步操作也会导致不必要的性能损耗。上面的append()方法就属于这类情况。如果虚拟机探测到有这样一串零碎的操作都对同一个对象加锁，将会把加锁同步的范围扩展（粗化）到整个操作序列的外部

# AQS

提供了一个基于FIFO队列，可以用于构建锁或者其他相关同步装置的基础框架。

在将AQS之前，我们先讲一下锁的套路：

1. 当多个线程过来竞争锁的时候，只有一个【可以有多个】能成功。

1. 获取到锁的线程会设置state为加锁状态，并设置当前线程为锁的持有者【可以支持锁重入】。【想想这是为什么】

1. 没有获取到锁的线程，就会被协调到队列中进行排队

1. 当锁线程释放锁之后，就会调度队列中的线程去枪锁【根据调度方式的不同，分为公平锁和非公平锁】

在上面的流程中有几个核心点：

- state:表示锁是否被持有。每个线程枪锁时，都会判断这个锁是否被持有，没有持有则占锁

- 锁的持有线程信息： 当锁被持有后，还有一种情况就是持有线程又来加锁了，此时可以比对这个信息来支持重入【还需要state计数重入次数】

- 排队队列【阻塞队列】：这个用来用来存储等待的线程，锁释放之后，会叫醒其中的线程抢锁。

明白这些以后，我们先看看怎么使用AQS，稍后再说源码。

下面的代码是一个不可重入的互斥锁，它使用值 0 表示解锁状态，使用值 1 表示锁定状态。 虽然不可重入锁并不严格要求记录当前所有者线程，建议这样做，因为更容易监控。 它还支持condition：

class Mutex implements Lock, java.io.Serializable {

  // Our internal helper class

  private static class Sync extends AbstractQueuedSynchronizer {

   // 获取锁

   public boolean tryAcquire(int acquires) {

​    assert acquires == 1; // Otherwise unused

​    if (compareAndSetState(0, 1)) {

​     setExclusiveOwnerThread(Thread.currentThread());

​     return true;

​    }

​    return false;

   }

   // 释放锁

   protected boolean tryRelease(int releases) {

​    assert releases == 1; // Otherwise unused

​    if (!isHeldExclusively()) //判断线程是否是锁的持有者

​     throw new IllegalMonitorStateException();

​    setExclusiveOwnerThread(null);

​    setState(0);

​    return true;

   }

   // 检测锁是否已经被占用

   public boolean isLocked() {

​    return getState() != 0;

   }

   // 检测当前线程是否是锁的持有者

   public boolean isHeldExclusively() {

​    // a data race, but safe due to out-of-thin-air guarantees

​    return getExclusiveOwnerThread() == Thread.currentThread();

   }

   //创建 Condition

   public Condition newCondition() {

​    return new ConditionObject();

   }

   // 反序列化

   private void readObject(ObjectInputStream s)

​     throws IOException, ClassNotFoundException {

​    s.defaultReadObject();

​    setState(0); // reset to unlocked state

   }

  }

  // The sync object does all the hard work. We just forward to it.

  private final Sync sync = new Sync();

  public void lock()        { sync.acquire(1); }

  public boolean tryLock()     { return sync.tryAcquire(1); }

  public void unlock()       { sync.release(1); }

  public Condition newCondition() { return sync.newCondition(); }

  public boolean isLocked()    { return sync.isLocked(); }

  public boolean isHeldByCurrentThread() {

   return sync.isHeldExclusively();

  }

  public boolean hasQueuedThreads() {

   return sync.hasQueuedThreads();

  }

  public void lockInterruptibly() throws InterruptedException {

   sync.acquireInterruptibly(1);

  }

  public boolean tryLock(long timeout, TimeUnit unit)

​    throws InterruptedException {

   return sync.tryAcquireNanos(1, unit.toNanos(timeout));

  }

 }

看完这些代码，你可能比较晕，里面提供了一些名字几乎相同的方法，比如release和tryRelease，我们先来解析一下这个。

我们先来看几个受保护的方法，需要子类去实现的：

- tryAcquire(int arg)：获取独占锁，成功返回true，失败返回false。该方法内会根据state值来决定返回结果

- tryRelease(int arg)：释放独占锁，成功返回true,失败返回false.

- tryAcquireShared(int arg)：获取共享锁，成功返回true，失败返回false。

- tryReleaseShared(int arg)：释放共享锁，成功返回true,失败返回false.

- isHeldExclusively：是否是独占锁。

这些方法需要子类继承并实现，才能提供服务。

下面我们来看AQS提供的对外公开方法：

- acquire(int arg)：获取独占锁，成功则返回，失败可能会阻塞

- acquireInterruptibly(int arg)：acquire的升级版，获取锁的过程支持可打断

- acquireShared(int arg)：获取共享锁，负数表示失败；0表示成功，但没有剩余可用资源；正数表示成功，且有剩余资源

- acquireSharedInterruptibly(int arg)：获取共享锁，过程支持可打断。

- release(int arg)：释放独占锁

- releaseShared(int arg)：释放共享锁，如果释放后允许唤醒后续等待结点返回true，否则返回false。

- tryAcquireNanos(int arg, long nanosTimeout)：获取独占锁，支持超时

- tryAcquireSharedNanos(int arg, long nanosTimeout)：获取共享锁，支持超时

这些方法提供了加锁的入口，我们应该先从这些方法的源码看起。

等待队列

​    

​     // CANCELLED，值为1，表示当前的线程被取消；

​     // SIGNAL，值为-1，表示当前节点的后继节点包含的线程需要运行，也就是unpark；

​     // CONDITION，值为-2，表示当前节点在等待condition，也就是在condition队列中；

​     // PROPAGATE，值为-3，表示当前场景下后续的acquireShared能够得以执行；

​     // 值为0，表示当前节点在sync队列中，等待着获取锁。

​     volatile int waitStatus; //等待的状态

​     volatile Node prev; //前一个节点

​     volatile Node next; //后一个节点

​     volatile Thread thread; //持有该节点的线程

​     Node nextWaiter; //存储condition队列中的后继节点。

​     Node() {   // Used to establish initial head or SHARED marker

​     }

​     Node(Thread thread, Node mode) {   // Used by addWaiter

​       this.nextWaiter = mode;

​       this.thread = thread;

​     }

​     Node(Thread thread, int waitStatus) { // Used by Condition

​       this.waitStatus = waitStatus;

​       this.thread = thread;

​     }

## 独占加锁流程【ReentrantLock】

代码顺序：lock>> acquire(1)>>tryAcquire(1)【ReentrantLock.Sync.nonfairTryAcquire(1)】>>addWaiter(Node.EXCLUSIVE), 1)>>acquireQueued

对应的流程：开始>>获取锁，成功则终止>>添加到等待队列>>再次尝试获取锁，失败则等待

【入口，尝试直接加锁，简化版】

​     final void lock() {

​       if (compareAndSetState(0, 1))

​         setExclusiveOwnerThread(Thread.currentThread());

​       else

​         acquire(1);

​     }

【流程概览】

   public final void acquire(int arg) { //arg=1

​     if (!tryAcquire(arg) && // 尝试获取锁，成功则返回

​       acquireQueued(addWaiter(Node.EXCLUSIVE), arg)) //添加到等待队列 ，Node.EXCLUSIVE是null,表示独占模式

​       selfInterrupt(); //该模式下不支持打断，这个是被打断线程获取到锁之后，根据之前的打断状态，然后自我打断

   }

【获取锁的过程，详细版】tryAcquire底层执行的是下面的代码：

​     final boolean nonfairTryAcquire(int acquires) { //acquires=1

​       final Thread current = Thread.currentThread();

​       int c = getState();  

​       if (c == 0) { // 0:未加锁

​         if (compareAndSetState(0, acquires)) { //加锁

​           setExclusiveOwnerThread(current);

​           return true;

​         }

​       }

​       else if (current == getExclusiveOwnerThread()) { //是否是锁重入

​         int nextc = c + acquires;

​         if (nextc < 0) // overflow //超过整数的范围了，重入的次数太多了

​           throw new Error("Maximum lock count exceeded");

​         setState(nextc); //锁重入计数

​         return true;

​       }

​       return false;

​     }

枪锁失败，【添加节点到阻塞队列】

   private Node addWaiter(Node mode) { //mode 是null

​     Node node = new Node(Thread.currentThread(), mode); //mode传入null，代表该节点不是用于await队列

​     // Try the fast path of enq; backup to full enq on failure

​     Node pred = tail;

​     if (pred != null) { //队列中有节点

​      node.prev = pred; //设置添加节点的前驱

​       if (compareAndSetTail(pred, node)) { //设置当前节点为尾节点

​        pred.next = node; //原先尾节点的next指向新的尾节点

​         return node;

​       }

​     }

​     // 初始化队列，并添加节点

​     enq(node);

​     return node;

   }

【初始化等待对立，并添加节点】

   private Node enq(final Node node) {

​     for (;;) {

​       Node t = tail;

​       if (t == null) { // 这是第一次进来，初始化队列，头尾相等，

​         if (compareAndSetHead(new Node())) //设置头节点

​          tail = head; //设置尾节点

​       } else { //第二次进来，将新节点添加到队列

​        node.prev = t;

​         if (compareAndSetTail(t, node)) {

​          t.next = node;

​           return t;

​         }

​       }

​     }

   }

【队列节点进入阻塞状态】

   final boolean acquireQueued(final Node node, int arg) { //node表示被添加的节点 arg=1

​     boolean failed = true;

​     try {

​       boolean interrupted = false; //线程阻塞的过程中，是否被打断过

​       for (;;) { //线程被唤醒也会走这里

​         final Node p = node.predecessor(); //前驱节点

​         //如果是队列中的第一个节点，阻塞前，再尝试获取一把锁

​         if (p == head && tryAcquire(arg)) {  

​           setHead(node); //将节点设置为头节点，原先头节点移除

​          p.next = null; // help GC

​          failed = false;

​           return interrupted;

​         }

​         // 检查受否应该让线程阻塞，通过设置节点的waitStatus

​         if (shouldParkAfterFailedAcquire(p, node) &&

​           parkAndCheckInterrupt())  //阻塞线程

​          interrupted = true; //线程被打断过则设置打断标记为true

​       }

​     } finally {

​       if (failed)

​         cancelAcquire(node);

​     }

   }

   private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) { // node.pre == pred

​     int ws = pred.waitStatus;

​     if (ws == Node.SIGNAL) //-1 ， 节点状态已经是Signal了，可以被阻塞 

​       return true;

​     if (ws > 0) { //前驱节点被取消了

​       do { //删除前驱节点

​        node.prev = pred = pred.prev;

​       } while (pred.waitStatus > 0);

​      pred.next = node;

​     } else {  //0或-3(PROPAGATE)的情况

​       // 设置节点为signal

​       compareAndSetWaitStatus(pred, ws, Node.SIGNAL);

​     }

​     return false;

   }

   // 阻塞线程了

   private final boolean parkAndCheckInterrupt() {

​     LockSupport.park(this);

​     // 判断线程是否被打断过，并清除打断标记

​     return Thread.interrupted();

   }

下面是线程A持有锁，线程B抢锁过程的示意图：

1. Thead-0获取到了锁

![img](https://vipkshttps8.wiz.cn/editor/63531d60-bd31-11ea-aa67-1778e2b2bbbd/20aa8caa-8757-4902-b032-92d733f8b349/resources/674-z2-6WcjZhCg-QJH0KE4P5-Va6VbgzVa42UYAzGk.png?token=W.hfyxpaKeZ1ALEo-GLY7TuIA8SX33YHKhnov6_ndm3LeF2E0ocVrbQuP2ZExmv08)

1. Thread-1 执行了CAS 尝试将 state 由 0 改为 1，结果失败。进入 tryAcquire 逻辑，这时 state 已经是1，结果仍然失败

![img](https://vipkshttps8.wiz.cn/editor/63531d60-bd31-11ea-aa67-1778e2b2bbbd/20aa8caa-8757-4902-b032-92d733f8b349/resources/_d7XwjkkjIFlEumUp_7WyuphGfy5HpWTon7Y3bIN79Y.png?token=W.hfyxpaKeZ1ALEo-GLY7TuIA8SX33YHKhnov6_ndm3LeF2E0ocVrbQuP2ZExmv08)

1. 接下来进入 addWaiter 逻辑，构造 Node 队列。图中黄色三角表示该 Node 的 waitStatus 状态，其中 0 为默认正常状态 。Node 的创建是懒惰的 。其中第一个 Node 称为 Dummy（哑元）或哨兵，用来占位，并不关联线程

![img](https://vipkshttps8.wiz.cn/editor/63531d60-bd31-11ea-aa67-1778e2b2bbbd/20aa8caa-8757-4902-b032-92d733f8b349/resources/LPf4WQFzWF7FLGYbP7Ihx-4_ZKKoSzDYaEHUfpKNGKk.png?token=W.hfyxpaKeZ1ALEo-GLY7TuIA8SX33YHKhnov6_ndm3LeF2E0ocVrbQuP2ZExmv08)

1. 当前线程进入 acquireQueued 逻辑 【acquireQueued 会在一个死循环中尝试几次获得锁，失败后进入 park 阻塞 】。如果自己是紧邻着 head（排第二位），那么再次 tryAcquire 尝试获取锁，当然这时 state 仍为 1，失败 。 进入 shouldParkAfterFailedAcquire 逻辑，将前驱 node，即 head 的 waitStatus 改为 -1，这次返回 false；

![img](https://vipkshttps8.wiz.cn/editor/63531d60-bd31-11ea-aa67-1778e2b2bbbd/20aa8caa-8757-4902-b032-92d733f8b349/resources/dtbDnvAWG9A6QTCjFki4GNrL8FSv8VNYg4pxM6yqiXE.png?token=W.hfyxpaKeZ1ALEo-GLY7TuIA8SX33YHKhnov6_ndm3LeF2E0ocVrbQuP2ZExmv08)

1.  shouldParkAfterFailedAcquire 执行完毕回到 acquireQueued ，再次 tryAcquire 尝试获取锁，当然这时state 仍为 1，失败。当再次进入 shouldParkAfterFailedAcquire 时，这时因为其前驱 node 的 waitStatus 已经是 -1，这次返回true。进入 parkAndCheckInterrupt， Thread-1 park（灰色表示）

![img](https://vipkshttps8.wiz.cn/editor/63531d60-bd31-11ea-aa67-1778e2b2bbbd/20aa8caa-8757-4902-b032-92d733f8b349/resources/TnquWZAIXSPsAP_Y8ThRudrSbZuwsGjUgl4U-1oyUI0.png?token=W.hfyxpaKeZ1ALEo-GLY7TuIA8SX33YHKhnov6_ndm3LeF2E0ocVrbQuP2ZExmv08)

1. 再次有多个线程经历上述过程竞争失败，变成这个样子：

![img](https://vipkshttps8.wiz.cn/editor/63531d60-bd31-11ea-aa67-1778e2b2bbbd/20aa8caa-8757-4902-b032-92d733f8b349/resources/xRn7Qv5aO3uFqAO1kVyPXX5IIHBYLsQAvMveRr1SAW4.png?token=W.hfyxpaKeZ1ALEo-GLY7TuIA8SX33YHKhnov6_ndm3LeF2E0ocVrbQuP2ZExmv08)

## 独占锁解锁流程

【释放锁的总流程】

   public final boolean release(int arg) {

​     if (tryRelease(arg)) { //解锁成功

​       Node h = head;

​       if (h != null && h.waitStatus != 0) //有排队等待的线程, -1 -2 -3 

​         unparkSuccessor(h); //唤醒线程 

​       return true;

​     }

​     return false;

   }

【释放锁的操作】

​     protected final boolean tryRelease(int releases) { //releases=1

​       int c = getState() - releases;

​       if (Thread.currentThread() != getExclusiveOwnerThread()) //必须是持有锁的线程才能释放锁

​         throw new IllegalMonitorStateException();

​       boolean free = false;

​       if (c == 0) { // c=0 ；表示锁都释放成功了，其他大于0的值，表示还有重入的锁

​        free = true;

​         setExclusiveOwnerThread(null);

​       }

​       setState(c); 

​       return free;

​     }

【唤醒其他线程来枪锁】

   private void unparkSuccessor(Node node) { //node是头节点

​     // 如果节点是负值，代表是需要唤醒后续节点

​     int ws = node.waitStatus;

​     if (ws < 0)

​       // 节点修改成0，该节点如果是线程节点，在唤醒后可能被删除了

​       compareAndSetWaitStatus(node, ws, 0);

​     // unpark后继节点，通常是下一个节点。 但如果下个节点的状态是取消或明显为空，

​     // 从尾部向后遍历以找到实际未取消的节点。

​     Node s = node.next;

​     if (s == null || s.waitStatus > 0) { // 1代表取消

​      s = null; 

​       for (Node t = tail; t != null && t != node; t = t.prev)

​         if (t.waitStatus <= 0)

​          s = t;  // 找到待唤醒的节点

​     }

​     if (s != null) //唤醒后续节点

​       LockSupport.unpark(s.thread);

   }

此时需要我们继续回到原先抢锁暂停的地方。

我们来总结一下这个过程：

1. Thread-0 释放锁，进入 tryRelease 流程，如果成功，设置 exclusiveOwnerThread 为 null，state = 0

![img](https://vipkshttps8.wiz.cn/editor/63531d60-bd31-11ea-aa67-1778e2b2bbbd/20aa8caa-8757-4902-b032-92d733f8b349/resources/Mt1UKR4hH_Z7VQG-3bFrngkq3l9V2J5CNH60eInHmFE.png?token=W.hfyxpaKeZ1ALEo-GLY7TuIA8SX33YHKhnov6_ndm3LeF2E0ocVrbQuP2ZExmv08)

1. 当前队列不为 null，并且 head 的 waitStatus = -1，进入 unparkSuccessor 流程，找到队列中离 head 最近的一个 Node（没取消的），unpark 恢复其运行，本例中即为 Thread-1回到 Thread-1 的 acquireQueued 流程：

![img](https://vipkshttps8.wiz.cn/editor/63531d60-bd31-11ea-aa67-1778e2b2bbbd/20aa8caa-8757-4902-b032-92d733f8b349/resources/tACwDnvGDEj7sfIRB4_5Bz6zmdHneXp7ESE8_JqH99I.png?token=W.hfyxpaKeZ1ALEo-GLY7TuIA8SX33YHKhnov6_ndm3LeF2E0ocVrbQuP2ZExmv08)

1. 如果加锁成功（没有竞争），会设置exclusiveOwnerThread 为 Thread-1，state = 1。如果这时候有其它线程来竞争（非公平的体现），例如这时有 Thread-4 来了：

![img](https://vipkshttps8.wiz.cn/editor/63531d60-bd31-11ea-aa67-1778e2b2bbbd/20aa8caa-8757-4902-b032-92d733f8b349/resources/KpOKK2_nqucCq3_-wv4r_x9oZknBPI6OSBWFWjiTmhQ.png?token=W.hfyxpaKeZ1ALEo-GLY7TuIA8SX33YHKhnov6_ndm3LeF2E0ocVrbQuP2ZExmv08)

1. 如果不巧又被 Thread-4 占了先，Thread-4 被设置为 exclusiveOwnerThread，state = 1。Thread-1 再次进入 acquireQueued 流程，获取锁失败，重新进入 park 阻塞

## 打断锁的原理

上面的代码中所讲的锁，是不可打断的。即使线程被打断，他仍然在队列节点中等待。当抢到锁之后，会根据线程的打断记录来进行一次自我打断的调用。

acquireInterruptibly(int arg)方法支持打断，我们来看一下原理：

   public final void acquireInterruptibly(int arg)

​       throws InterruptedException {

​     if (Thread.interrupted())

​       throw new InterruptedException();

​     if (!tryAcquire(arg)) // 尝试获取锁，成功则终止

​       doAcquireInterruptibly(arg);

   }

下面就是加入阻塞队列的流程了，跟上面的几乎一样：

   private void doAcquireInterruptibly(int arg)

​     throws InterruptedException {

​     final Node node = addWaiter(Node.EXCLUSIVE);

​     boolean failed = true;

​     try {

​       for (;;) {

​         final Node p = node.predecessor();

​         if (p == head && tryAcquire(arg)) {

​           setHead(node);

​          p.next = null; // help GC

​          failed = false;

​           return;

​         }

​         if (shouldParkAfterFailedAcquire(p, node) &&

​           parkAndCheckInterrupt()) // 被打断了

​           throw new InterruptedException(); //直接抛出异常

​       }

​     } finally {

​       if (failed)

​         cancelAcquire(node);

​     }

   }

## 公平锁的实现

新线程每次在抢锁的时候，会检查队列中是否有等待的线程，有就去排队

​     protected final boolean tryAcquire(int acquires) {

​       final Thread current = Thread.currentThread();

​       int c = getState();

​       if (c == 0) {

​         // 检查是否有排队的线程

​         if (!hasQueuedPredecessors() &&

​           compareAndSetState(0, acquires)) {

​           setExclusiveOwnerThread(current);

​           return true;

​         }

​       }

​       else if (current == getExclusiveOwnerThread()) {

​         int nextc = c + acquires;

​         if (nextc < 0)

​           throw new Error("Maximum lock count exceeded");

​         setState(nextc);

​         return true;

​       }

​       return false;

​     }

   public final boolean hasQueuedPredecessors() {

​     // The correctness of this depends on head being initialized

​     // before tail and on head.next being accurate if the current

​     // thread is first in queue.

​     Node t = tail; // Read fields in reverse initialization order

​     Node h = head;

​     Node s;

​     return h != t &&

​       ((s = h.next) == null || s.thread != Thread.currentThread());

   }

## 锁超时

流程基本跟可打算锁一样，在实现的park的时候，增加了超时时间，这样线程可以在超时之后醒来，并终止获取锁流程。

   public final boolean tryAcquireNanos(int arg, long nanosTimeout)

​       throws InterruptedException {

​     if (Thread.interrupted())

​       throw new InterruptedException();

​     return tryAcquire(arg) ||

​       doAcquireNanos(arg, nanosTimeout); 

   }

   private boolean doAcquireNanos(int arg, long nanosTimeout)

​       throws InterruptedException {

​     if (nanosTimeout <= 0L)

​       return false;

​     final long deadline = System.nanoTime() + nanosTimeout;

​     final Node node = addWaiter(Node.EXCLUSIVE);

​     boolean failed = true;

​     try {

​       for (;;) {

​         final Node p = node.predecessor();

​         if (p == head && tryAcquire(arg)) {

​           setHead(node);

​          p.next = null; // help GC

​          failed = false;

​           return true;

​         }

​        nanosTimeout = deadline - System.nanoTime();

​         if (nanosTimeout <= 0L)

​           return false;

​         if (shouldParkAfterFailedAcquire(p, node) &&

​          nanosTimeout > spinForTimeoutThreshold)

​           LockSupport.parkNanos(this, nanosTimeout);

​         if (Thread.interrupted())

​           throw new InterruptedException();

​       }

​     } finally {

​       if (failed)  // 超时之后要执行一些清理工作

​         cancelAcquire(node);

​     }

   }

## 条件变量的实现原理

1. 如果当前线程被中断，则抛出 InterruptedException。

1. 保存由 getState 返回的锁状态。

1. 以保存的状态作为参数调用 release，如果失败则抛出 IllegalMonitorStateException。

1. 阻塞直到发出信号或被中断。

1. 通过以保存的状态作为参数调用特定版本的获取来重新获取。

1. 如果在步骤 4 中被阻塞时被中断，则抛出 InterruptedException。

​     public final void await() throws InterruptedException {

​       if (Thread.interrupted())

​         throw new InterruptedException();

​       // 添加到等待队列

​       Node node = addConditionWaiter();

​       // 释放锁

​       int savedState = fullyRelease(node);

​       int interruptMode = 0;

​       // 判断节点是否已经被移动到阻塞队列，有可能刚调用await,另外一个线程调用了signal

​       while (!isOnSyncQueue(node)) { 

​         LockSupport.park(this); //进入阻塞模式

​         if ((interruptMode = checkInterruptWhileWaiting(node)) != 0) //检测节点在等待过程是否被打断

​           break;

​       }

​       // 被唤醒之后，添加到阻塞队列

​       if (acquireQueued(node, savedState) && interruptMode != THROW_IE)

​        interruptMode = REINTERRUPT;

​       if (node.nextWaiter != null) 

​         unlinkCancelledWaiters(); // 清除失效的节点

​       if (interruptMode != 0)

​         reportInterruptAfterWait(interruptMode);

​     }

​     // 检测线程是否被打断

​     private int checkInterruptWhileWaiting(Node node) {

​       return Thread.interrupted() ?

​         (transferAfterCancelledWait(node) ? THROW_IE : REINTERRUPT) :

​         0;

​     }

​     // 根据打断模式进行相应的处理

​     private void reportInterruptAfterWait(int interruptMode)

​       throws InterruptedException {

​       if (interruptMode == THROW_IE)

​         throw new InterruptedException();

​       else if (interruptMode == REINTERRUPT)

​         selfInterrupt();

​     }

  

   final int fullyRelease(Node node) {

​     boolean failed = true;

​     try {

​       int savedState = getState();

​       if (release(savedState)) {

​        failed = false;

​         return savedState;

​       } else {

​         throw new IllegalMonitorStateException();

​       }

​     } finally {

​       if (failed)

​        node.waitStatus = Node.CANCELLED;

​     }

   }

   final boolean isOnSyncQueue(Node node) {

​     if (node.waitStatus == Node.CONDITION || node.prev == null)

​       return false;

​     if (node.next != null) //如果有后继节点，肯定在同步队列

​       return true;

   

​     return findNodeFromTail(node);

   }

   // 从阻塞队列中查找该节点，true表示在阻塞队列

   private boolean findNodeFromTail(Node node) {

​     Node t = tail;

​     for (;;) {

​       if (t == node)

​         return true;

​       if (t == null)

​         return false;

​      t = t.prev;

​     }

   }

【添加等待节点】

​     private Node addConditionWaiter() {

​       Node t = lastWaiter;

​       // 取消失效的等待节点

​       if (t != null && t.waitStatus != Node.CONDITION) {

​         unlinkCancelledWaiters();

​        t = lastWaiter;

​       }

​       //下面的流程就是创建节点并添加到队列中

​       Node node = new Node(Thread.currentThread(), Node.CONDITION);

​       if (t == null)

​        firstWaiter = node;

​       else

​        t.nextWaiter = node;

​      lastWaiter = node;

​       return node;

​     }

### 唤醒机制

​     public final void signal() {

​       if (!isHeldExclusively())

​         throw new IllegalMonitorStateException();

​       Node first = firstWaiter;

​       if (first != null)

​         doSignal(first);

​     }

​     private void doSignal(Node first) {

​       do {

​         if ( (firstWaiter = first.nextWaiter) == null)

​          lastWaiter = null;

​        first.nextWaiter = null;

​       } while (!transferForSignal(first) &&

​           (first = firstWaiter) != null);

​     }

   final boolean transferForSignal(Node node) {

​     // 如果设置不成功，说明切点已经被取消

​     if (!compareAndSetWaitStatus(node, Node.CONDITION, 0))

​       return false;

​     // 节点入队，p是当前节点的前驱

​     Node p = enq(node);

​     int ws = p.waitStatus;  

​    // 前驱节点需要设置waitstatus=-1,表示他的后继节点需要被唤醒

​     if (ws > 0 || !compareAndSetWaitStatus(p, ws, Node.SIGNAL))

​       LockSupport.unpark(node.thread);

​     return true;

   }

下面我们来对这个代码流程进行总结：

1. 开始 Thread-0 持有锁，调用 await，进入 ConditionObject 的 addConditionWaiter 流程。创建新的 Node 状态为 -2（Node.CONDITION），关联 Thread-0，加入等待队列尾部。

![img](https://vipkshttps8.wiz.cn/editor/63531d60-bd31-11ea-aa67-1778e2b2bbbd/20aa8caa-8757-4902-b032-92d733f8b349/resources/R5ME5HM2JLIUrbY-cZY7xFnQIr68Fq90GT_lt9oHTO0.png?token=W.hfyxpaKeZ1ALEo-GLY7TuIA8SX33YHKhnov6_ndm3LeF2E0ocVrbQuP2ZExmv08)

1. 接下来进入 AQS 的 fullyRelease 流程，释放同步器上的锁

![img](https://vipkshttps8.wiz.cn/editor/63531d60-bd31-11ea-aa67-1778e2b2bbbd/20aa8caa-8757-4902-b032-92d733f8b349/resources/Mm3U23PjMwPdo2S_4KxODPfAzxNAr3SyQ7auqcFJB24.png?token=W.hfyxpaKeZ1ALEo-GLY7TuIA8SX33YHKhnov6_ndm3LeF2E0ocVrbQuP2ZExmv08)

1. unpark AQS 队列中的下一个节点，竞争锁，假设没有其他竞争线程，那么 Thread-1 竞争成功

![img](https://vipkshttps8.wiz.cn/editor/63531d60-bd31-11ea-aa67-1778e2b2bbbd/20aa8caa-8757-4902-b032-92d733f8b349/resources/vcJBjn8znJnIN4tPdnoR5C0OhXeVNEk65mctU2NpSZk.png?token=W.hfyxpaKeZ1ALEo-GLY7TuIA8SX33YHKhnov6_ndm3LeF2E0ocVrbQuP2ZExmv08)

1. park 阻塞 Thread-0：

![img](https://vipkshttps8.wiz.cn/editor/63531d60-bd31-11ea-aa67-1778e2b2bbbd/20aa8caa-8757-4902-b032-92d733f8b349/resources/u1CjQTNu9IYKiMy3YDsrr1hU-B170F_l6amaXIIrBnY.png?token=W.hfyxpaKeZ1ALEo-GLY7TuIA8SX33YHKhnov6_ndm3LeF2E0ocVrbQuP2ZExmv08)

1. 下面进入唤醒流程，假设 Thread-1 要来唤醒 Thread-0，进入 ConditionObject 的 doSignal 流程，取得等待队列中第一个 Node，即 Thread-0 所在 Node

![img](https://vipkshttps8.wiz.cn/editor/63531d60-bd31-11ea-aa67-1778e2b2bbbd/20aa8caa-8757-4902-b032-92d733f8b349/resources/nYSEMuThjEXWtF1TohQ94p7GHTIfX-_1EpXAnMz9EdI.png?token=W.hfyxpaKeZ1ALEo-GLY7TuIA8SX33YHKhnov6_ndm3LeF2E0ocVrbQuP2ZExmv08)

1. 执行 transferForSignal 流程，将该 Node 加入 AQS 队列尾部，将 Thread-0 的 waitStatus 改为 0，Thread-3 的waitStatus 改为 -1

![img](https://vipkshttps8.wiz.cn/editor/63531d60-bd31-11ea-aa67-1778e2b2bbbd/20aa8caa-8757-4902-b032-92d733f8b349/resources/CGIzn_1AdHAPJEQbB9BXP-MlJ5VCR9PTXQ4bQWFIKrE.png?token=W.hfyxpaKeZ1ALEo-GLY7TuIA8SX33YHKhnov6_ndm3LeF2E0ocVrbQuP2ZExmv08)

1. Thread-1 释放锁，进入 unlock 流程，略。

## 读写锁

读写锁的应用场景：【更新缓存】

 class CachedData {

  Object data;

  volatile boolean cacheValid;

  final ReentrantReadWriteLock rwl = new ReentrantReadWriteLock();

  void processCachedData() {

   rwl.readLock().lock();

   if (!cacheValid) {

​    // 获取写锁之前必须释放读锁

​    rwl.readLock().unlock();

​    rwl.writeLock().lock();

​    try {

​     // 重新检查，可能会有别的写线程已经修改了数据

​     if (!cacheValid) {

​      data = ...

​      cacheValid = true;

​     }

​     // 在释放写锁前，先降级为读锁

​     rwl.readLock().lock();

​    } finally {

​     rwl.writeLock().unlock(); // 释放了写锁，但是仍然持有读锁

​    }

   }

   try {

​    use(data);

   } finally {

​    rwl.readLock().unlock();

   }

  }

 }

在看源代码之前，我们先来明确几点：

- 读写锁用的是同一个 Sycn 同步器，因此等待队列、state 等也是同一个

- 流程与 ReentrantLock 加锁相比没有特殊之处，不同是写锁状态占了 state 的低 16 位，而读锁使用的是 state 的高 16 位

- 上写锁成功之后，后序的节点也会依次加入到阻塞队列中。不过，在写锁释放的时候，会唤醒所有的读锁【假如后面紧跟者着多个读锁】

先看一下锁的构建函数,底层使用的是同一个同步器：

   public ReentrantReadWriteLock(boolean fair) {

​    sync = fair ? new FairSync() : new NonfairSync();

​    readerLock = new ReadLock(this);

​    writerLock = new WriteLock(this);

   }

​     protected ReadLock(ReentrantReadWriteLock lock) {

​      sync = lock.sync;

​     }

​     protected WriteLock(ReentrantReadWriteLock lock) {

​      sync = lock.sync;

​     }

### 写锁加锁

   // 下面这些代码是不是很熟悉，关键看子类实现的tryAcquire方法

   public void lock() {

​      sync.acquire(1);

   }

   public final void acquire(int arg) {

​     if (!tryAcquire(arg) &&

​       acquireQueued(addWaiter(Node.EXCLUSIVE), arg))

​       selfInterrupt();

   }

【写锁加锁】:

1. （如果读锁计数非零）||（写锁计数非零且不是当前线程），写锁获取失败

1.  如果锁计数超过限定的范围，则抛出异常

1. 否则，此线程有资格锁定， 并更新状态并设置所有者。

​     protected final boolean tryAcquire(int acquires) {

​       Thread current = Thread.currentThread();

​       int c = getState();

​       int w = exclusiveCount(c); //获取写锁上的计数

​       if (c != 0) {

​         //写锁计数不为零，并且非当前线程，则获取锁失败

​         if (w == 0 || current != getExclusiveOwnerThread())

​           return false;

​         if (w + exclusiveCount(acquires) > MAX_COUNT)

​           throw new Error("Maximum lock count exceeded");

​         // Reentrant acquire

​         setState(c + acquires);

​         return true;

​       }

​       // writerShouldBlock：用来判断是否是公平锁，非公平默认返回true

​       if (writerShouldBlock() ||

​       //cas 更新锁计数，成功则返回true,表示加锁成功

​         !compareAndSetState(c, c + acquires)) 

​         return false;

​       setExclusiveOwnerThread(current); 

​       return true;

​     }

假如现在写锁加锁成功了，还没有释放锁，我们来看读锁的工作流程。

   public void lock() {

​      sync.acquireShared(1);

​     }

   public final void acquireShared(int arg) { // arg=1

​     // -1 表示失败

​     // 0 表示成功，但后继节点不会继续唤醒

​    // 正数表示成功，而且数值是还有几个后继节点需要唤醒，读写锁返回 1

​     if (tryAcquireShared(arg) < 0)  // 尝试获取锁，成功则返回1，失败则返回-1

​       doAcquireShared(arg); //进入阻塞队列

   }

### 共享锁获取流程

1. 如果其他线程持有写锁，则失败，返回-1

1. cas更新读锁计数

1. 如果失败，进入完整版本的获取锁流程，该流程可能会让线程进入等待队列

​    protected final int tryAcquireShared(int unused) {

​       Thread current = Thread.currentThread();

​       int c = getState();

​       // 判断是否有写线程上锁，并且写线程不是当前线程

​       if (exclusiveCount(c) != 0 &&  

​         getExclusiveOwnerThread() != current)

​         return -1;

​       int r = sharedCount(c);

​       if (!readerShouldBlock() &&

​        r < MAX_COUNT &&

​         compareAndSetState(c, c + SHARED_UNIT)) { // cas抢锁

​         if (r == 0) {

​          firstReader = current;

​          firstReaderHoldCount = 1;

​         } else if (firstReader == current) {

​          firstReaderHoldCount++;

​         } else {

​           HoldCounter rh = cachedHoldCounter;

​           if (rh == null || rh.tid != getThreadId(current))

​            cachedHoldCounter = rh = readHolds.get();

​           else if (rh.count == 0)

​            readHolds.set(rh);

​          rh.count++;

​         }

​         return 1;  // 获得了读锁

​       }

​       return fullTryAcquireShared(current); // 完成版的抢锁流程

​     }

【完整版的读锁获取流程】，比较难懂，掠过

​     final int fullTryAcquireShared(Thread current) {

​       HoldCounter rh = null;

​       for (;;) {

​         int c = getState();

​         if (exclusiveCount(c) != 0) { //判断写锁计数

​           if (getExclusiveOwnerThread() != current) //判断写锁线程是否是当前线程

​             return -1;

​           // else we hold the exclusive lock; blocking here

​           // would cause deadlock.

​         } else if (readerShouldBlock()) {

​           // Make sure we're not acquiring read lock reentrantly

​           if (firstReader == current) {

​             // assert firstReaderHoldCount > 0;

​           } else {

​             if (rh == null) {

​              rh = cachedHoldCounter;

​               if (rh == null || rh.tid != getThreadId(current)) {

​                rh = readHolds.get();

​                 if (rh.count == 0)

​                  readHolds.remove();

​               }

​             }

​             if (rh.count == 0)

​               return -1;

​           }

​         }

​         if (sharedCount(c) == MAX_COUNT)

​           throw new Error("Maximum lock count exceeded");

​         if (compareAndSetState(c, c + SHARED_UNIT)) {

​           if (sharedCount(c) == 0) {

​            firstReader = current;

​            firstReaderHoldCount = 1;

​           } else if (firstReader == current) {

​            firstReaderHoldCount++;

​           } else {

​             if (rh == null)

​              rh = cachedHoldCounter;

​             if (rh == null || rh.tid != getThreadId(current))

​              rh = readHolds.get();

​             else if (rh.count == 0)

​              readHolds.set(rh);

​            rh.count++;

​            cachedHoldCounter = rh; // cache for release

​           }

​           return 1;

​         }

​       }

​     }

【读锁等待流程】

   private void doAcquireShared(int arg) {

​     // 添加等待节点到队尾，注意这里是共享节点，不是独占了

​     final Node node = addWaiter(Node.SHARED);

​     boolean failed = true;

​     try {

​       boolean interrupted = false;

​       for (;;) {

​         final Node p = node.predecessor();

​         if (p == head) { //如果当前节点的前驱是头节点，再次尝试获取锁

​           int r = tryAcquireShared(arg);

​           if (r >= 0) { //获取锁成功

​             setHeadAndPropagate(node, r); //继续唤醒后面的读锁

​            p.next = null; // help GC

​             if (interrupted)

​               selfInterrupt();

​            failed = false;

​             return;

​           }

​         }

​         // 检查线程是否需要阻塞

​         if (shouldParkAfterFailedAcquire(p, node) &&

​           parkAndCheckInterrupt()) //阻塞线程

​          interrupted = true;

​       }

​     } finally {

​       if (failed)

​         cancelAcquire(node);

​     }

   }

我们接着来看，读锁唤醒的流程setHeadAndPropagate：

 // node是已经唤醒的节点，propagate待唤醒的读锁个数

 private void setHeadAndPropagate(Node node, int propagate) {

​     Node h = head; // Record old head for check below

​     setHead(node);

​     if (propagate > 0 || h == null || h.waitStatus < 0 ||

​       (h = head) == null || h.waitStatus < 0) {

​       Node s = node.next;

​       if (s == null || s.isShared()) //后续节点还是共享节点，继续唤醒

​         doReleaseShared();  

​     }

   }

   private void doReleaseShared() {

​     for (;;) {

​       Node h = head;

​       if (h != null && h != tail) {

​         int ws = h.waitStatus;

​         if (ws == Node.SIGNAL) { // 节点需要唤醒

​           if (!compareAndSetWaitStatus(h, Node.SIGNAL, 0)) //设置头节点状态为0，防止乱序唤醒

​             continue;       // loop to recheck cases

​           unparkSuccessor(h);  

​         }

​         else if (ws == 0 &&

​             !compareAndSetWaitStatus(h, 0, Node.PROPAGATE))

​           continue;         // loop on failed CAS

​       }

​       if (h == head)          // loop if head changed

​         break;

​     }

   }

### 写锁的释放流程

   public final boolean release(int arg) {

​     if (tryRelease(arg)) {

​       Node h = head;

​       if (h != null && h.waitStatus != 0)

​         unparkSuccessor(h);

​       return true;

​     }

​     return false;

   }

  protected final boolean tryRelease(int releases) {

​       if (!isHeldExclusively())

​         throw new IllegalMonitorStateException();

​       int nextc = getState() - releases;

​       boolean free = exclusiveCount(nextc) == 0;

​       if (free)

​         setExclusiveOwnerThread(null);

​       setState(nextc);

​       return free;

 }

### 读锁的释放流程

   public final boolean releaseShared(int arg) {

​     if (tryReleaseShared(arg)) {

​       doReleaseShared();

​       return true;

​     }

​     return false;

   }

​     protected final boolean tryReleaseShared(int unused) {

​       Thread current = Thread.currentThread();

​       if (firstReader == current) {

​         // assert firstReaderHoldCount > 0;

​         if (firstReaderHoldCount == 1)

​          firstReader = null;

​         else

​          firstReaderHoldCount--;

​       } else {

​         HoldCounter rh = cachedHoldCounter;

​         if (rh == null || rh.tid != getThreadId(current))

​          rh = readHolds.get();

​         int count = rh.count;

​         if (count <= 1) {

​          readHolds.remove();

​           if (count <= 0)

​             throw unmatchedUnlockException();

​         }

​         --rh.count;

​       }

​       for (;;) {

​         int c = getState();

​         int nextc = c - SHARED_UNIT;

​         if (compareAndSetState(c, nextc))

​           // Releasing the read lock has no effect on readers,

​           // but it may allow waiting writers to proceed if

​           // both read and write locks are now free.

​           return nextc == 0;

​       }

​     }

我们来对上面的代码进行一下总结：假设 t1 w.lock，t2 r.lock

1.  t1 成功上锁，流程与 ReentrantLock 加锁相比没有特殊之处，不同是写锁状态占了 state 的低 16 位，而读锁使用的是 state 的高 16 位

![img](https://vipkshttps8.wiz.cn/editor/63531d60-bd31-11ea-aa67-1778e2b2bbbd/20aa8caa-8757-4902-b032-92d733f8b349/resources/cxOaEsgkZXHe_m67-5ltqO2_QNsHs7ChKaWrvxe4o2U.png?token=W.hfyxpaKeZ1ALEo-GLY7TuIA8SX33YHKhnov6_ndm3LeF2E0ocVrbQuP2ZExmv08)

1. t2 执行 r.lock，这时进入读锁的 sync.acquireShared(1) 流程，首先会进入 tryAcquireShared 流程。如果有写锁占据，那么 tryAcquireShared 返回 -1 表示失败

![img](https://vipkshttps8.wiz.cn/editor/63531d60-bd31-11ea-aa67-1778e2b2bbbd/20aa8caa-8757-4902-b032-92d733f8b349/resources/ZIoRy6PX6LOyhTZlHYYBcIat-cisbPK4jIZWdW1UXjM.png?token=W.hfyxpaKeZ1ALEo-GLY7TuIA8SX33YHKhnov6_ndm3LeF2E0ocVrbQuP2ZExmv08)

1. 这时会进入 sync.doAcquireShared(1) 流程，首先也是调用 addWaiter 添加节点，不同之处在于节点被设置为Node.SHARED 模式而非 Node.EXCLUSIVE 模式，注意此时 t2 仍处于活跃状态

![img](https://vipkshttps8.wiz.cn/editor/63531d60-bd31-11ea-aa67-1778e2b2bbbd/20aa8caa-8757-4902-b032-92d733f8b349/resources/0h08FJS_BJs-p79ki1HKDifY72TVO2gAhFELDlMxYaM.png?token=W.hfyxpaKeZ1ALEo-GLY7TuIA8SX33YHKhnov6_ndm3LeF2E0ocVrbQuP2ZExmv08)

1. t2 会看看自己的节点是不是老二，如果是，还会再次调用 tryAcquireShared(1) 来尝试获取锁

1. 如果没有成功，在 doAcquireShared 内 for (;;) 循环一次，把前驱节点的 waitStatus 改为 -1，再 for (;;) 循环一次尝试 tryAcquireShared(1) 如果还不成功，那么在 parkAndCheckInterrupt() 处 park

![img](https://vipkshttps8.wiz.cn/editor/63531d60-bd31-11ea-aa67-1778e2b2bbbd/20aa8caa-8757-4902-b032-92d733f8b349/resources/ER2u-5QWismu3NsS1chwhCPUe2XD3kksVSYPYFt8wak.png?token=W.hfyxpaKeZ1ALEo-GLY7TuIA8SX33YHKhnov6_ndm3LeF2E0ocVrbQuP2ZExmv08)

1. 这种状态下，假设又有 t3 加读锁和 t4 加写锁，这期间 t1 仍然持有锁，就变成了下面的样子

![img](https://vipkshttps8.wiz.cn/editor/63531d60-bd31-11ea-aa67-1778e2b2bbbd/20aa8caa-8757-4902-b032-92d733f8b349/resources/IVcISPms5F_z23sv7HQEVnZwl3Vv9yAPZt8dpRcZjw0.png?token=W.hfyxpaKeZ1ALEo-GLY7TuIA8SX33YHKhnov6_ndm3LeF2E0ocVrbQuP2ZExmv08)

1. t1 w.unlock这时会走到写锁的 sync.release(1) 流程，调用 sync.tryRelease(1) 成功，变成下面的样子

![img](https://vipkshttps8.wiz.cn/editor/63531d60-bd31-11ea-aa67-1778e2b2bbbd/20aa8caa-8757-4902-b032-92d733f8b349/resources/IeO4uDVy6UQf5bE_i-tPiHP7gzrN3f8VZxryq701qA4.png?token=W.hfyxpaKeZ1ALEo-GLY7TuIA8SX33YHKhnov6_ndm3LeF2E0ocVrbQuP2ZExmv08)

1. 接下来执行唤醒流程 sync.unparkSuccessor，即让老二恢复运行，这时 t2 在 doAcquireShared 内parkAndCheckInterrupt() 处恢复运行这回再来一次 for (;;) 执行 tryAcquireShared 成功则让读锁计数加一

![img](https://vipkshttps8.wiz.cn/editor/63531d60-bd31-11ea-aa67-1778e2b2bbbd/20aa8caa-8757-4902-b032-92d733f8b349/resources/F9PCjCLz6gZUdhHj4AaTNxVnlOkyFdLrzYYF8YBl8J4.png?token=W.hfyxpaKeZ1ALEo-GLY7TuIA8SX33YHKhnov6_ndm3LeF2E0ocVrbQuP2ZExmv08)

1. 这时 t2 已经恢复运行，接下来 t2 调用 setHeadAndPropagate(node, 1)，它原本所在节点被置为头节点

![img](https://vipkshttps8.wiz.cn/editor/63531d60-bd31-11ea-aa67-1778e2b2bbbd/20aa8caa-8757-4902-b032-92d733f8b349/resources/YcMTbw_vciUYa-02BFXV2LaRF8QDvTepkr38V7jo8IA.png?token=W.hfyxpaKeZ1ALEo-GLY7TuIA8SX33YHKhnov6_ndm3LeF2E0ocVrbQuP2ZExmv08)

事情还没完，在 setHeadAndPropagate 方法内还会检查下一个节点是否是 shared，如果是则调用doReleaseShared() 将 head 的状态从 -1 改为 0 并唤醒老二，这时 t3 在 doAcquireShared 内parkAndCheckInterrupt() 处恢复运行

![img](https://vipkshttps8.wiz.cn/editor/63531d60-bd31-11ea-aa67-1778e2b2bbbd/20aa8caa-8757-4902-b032-92d733f8b349/resources/bpAD4b0C8pzFv2qCQJHJS-6LidAPjL3B1lcfHL2smns.png?token=W.hfyxpaKeZ1ALEo-GLY7TuIA8SX33YHKhnov6_ndm3LeF2E0ocVrbQuP2ZExmv08)

1. 这回再来一次 for (;;) 执行 tryAcquireShared 成功则让读锁计数加一

![img](https://vipkshttps8.wiz.cn/editor/63531d60-bd31-11ea-aa67-1778e2b2bbbd/20aa8caa-8757-4902-b032-92d733f8b349/resources/1Pp_iqnbpFxkWw6xw4a7hhMcvkU4vgJMoJ_ZryBeLfE.png?token=W.hfyxpaKeZ1ALEo-GLY7TuIA8SX33YHKhnov6_ndm3LeF2E0ocVrbQuP2ZExmv08)

1. 这时 t3 已经恢复运行，接下来 t3 调用 setHeadAndPropagate(node, 1)，它原本所在节点被置为头节点

![img](https://vipkshttps8.wiz.cn/editor/63531d60-bd31-11ea-aa67-1778e2b2bbbd/20aa8caa-8757-4902-b032-92d733f8b349/resources/DmUixlBLbLhN8N_FSlbaaD3laQiOXpVYzoxlzygCDvs.png?token=W.hfyxpaKeZ1ALEo-GLY7TuIA8SX33YHKhnov6_ndm3LeF2E0ocVrbQuP2ZExmv08)

1. 下一个节点不是 shared 了，因此不会继续唤醒 t4 所在节点。

1. t2 r.unlock，t3 r.unlock。t2 进入 sync.releaseShared(1) 中，调用 tryReleaseShared(1) 让计数减一，但由于计数还不为零

![img](https://vipkshttps8.wiz.cn/editor/63531d60-bd31-11ea-aa67-1778e2b2bbbd/20aa8caa-8757-4902-b032-92d733f8b349/resources/W6NoQRYzUDOZ5V8Uvh5pCtGOeohBILAwr3816IHub3U.png?token=W.hfyxpaKeZ1ALEo-GLY7TuIA8SX33YHKhnov6_ndm3LeF2E0ocVrbQuP2ZExmv08)

1. t3 进入 sync.releaseShared(1) 中，调用 tryReleaseShared(1) 让计数减一，这回计数为零了，进入doReleaseShared() 将头节点从 -1 改为 0 并唤醒老二，即

![img](https://vipkshttps8.wiz.cn/editor/63531d60-bd31-11ea-aa67-1778e2b2bbbd/20aa8caa-8757-4902-b032-92d733f8b349/resources/5EdEifmIRetQ9JjA19xr8ftyuKsd_ks1MqHvOr63J8w.png?token=W.hfyxpaKeZ1ALEo-GLY7TuIA8SX33YHKhnov6_ndm3LeF2E0ocVrbQuP2ZExmv08)

1. 之后 t4 在 acquireQueued 中 parkAndCheckInterrupt 处恢复运行，再次 for (;;) 这次自己是老二，并且没有其他竞争，tryAcquire(1) 成功，修改头结点，流程结束

![img](https://vipkshttps8.wiz.cn/editor/63531d60-bd31-11ea-aa67-1778e2b2bbbd/20aa8caa-8757-4902-b032-92d733f8b349/resources/6rX9FI_lclHF8zqhuHrGRj6cqbK51S47SjmwpFoxI4U.png?token=W.hfyxpaKeZ1ALEo-GLY7TuIA8SX33YHKhnov6_ndm3LeF2E0ocVrbQuP2ZExmv08)

# Semaphore原理

信号量，用来限制能同时访问共享资源的线程上限。

Semaphore 有点像一个停车场，permits 就好像停车位数量，当线程获得了 permits 就像是获得了停车位，然后停车场显示空余车位减一

模拟：刚开始，permits（state）为 3，这时 5 个线程来获取资源

![img](https://vipkshttps8.wiz.cn/editor/63531d60-bd31-11ea-aa67-1778e2b2bbbd/20aa8caa-8757-4902-b032-92d733f8b349/resources/Z_rqI7QLkhf187olM8D6B3UKMXJ7Ak_a-dnEMNEI8Yo.png?token=W.hfyxpaKeZ1ALEo-GLY7TuIA8SX33YHKhnov6_ndm3LeF2E0ocVrbQuP2ZExmv08)

假设其中 Thread-1，Thread-2，Thread-4 cas 竞争成功，而 Thread-0 和 Thread-3 竞争失败，进入 AQS 队列park 阻塞

![img](https://vipkshttps8.wiz.cn/editor/63531d60-bd31-11ea-aa67-1778e2b2bbbd/20aa8caa-8757-4902-b032-92d733f8b349/resources/B-aob5MasR4iH_o8VUJK7qyfYoL7FB5Ktwz5zkvlbMw.png?token=W.hfyxpaKeZ1ALEo-GLY7TuIA8SX33YHKhnov6_ndm3LeF2E0ocVrbQuP2ZExmv08)

这时 Thread-4 释放了 permits，状态如下：

![img](https://vipkshttps8.wiz.cn/editor/63531d60-bd31-11ea-aa67-1778e2b2bbbd/20aa8caa-8757-4902-b032-92d733f8b349/resources/q1R0bW4SjUQKkPNSeOIDexjAU46R5nzOdsqZyiwEZME.png?token=W.hfyxpaKeZ1ALEo-GLY7TuIA8SX33YHKhnov6_ndm3LeF2E0ocVrbQuP2ZExmv08)

接下来 Thread-0 竞争成功，permits 再次设置为 0，设置自己为 head 节点，断开原来的 head 节点，unpark 接下来的 Thread-3 节点，但由于 permits 是 0，因此 Thread-3 在尝试不成功后再次进入 park 状态

![img](https://vipkshttps8.wiz.cn/editor/63531d60-bd31-11ea-aa67-1778e2b2bbbd/20aa8caa-8757-4902-b032-92d733f8b349/resources/Za6PAAZRHQAQams18dzuxJDiaIaJShzmBwiyyeKGVzc.png?token=W.hfyxpaKeZ1ALEo-GLY7TuIA8SX33YHKhnov6_ndm3LeF2E0ocVrbQuP2ZExmv08)

我们先来看怎么使用

​     Semaphore semaphore = new Semaphore(3);

​     for (int i = 0; i < 5; i++) {

​       new Thread(() -> {

​         try {

​          semaphore.acquire();

​           System.err.println(Thread.currentThread().getName() + "执行了");

​         } catch (InterruptedException e) {

​          e.printStackTrace();

​         } finally {

​          semaphore.release();

​         }

​       }).start();

​     }

看源码的顺序： new >> acquire >> release

先看如何构造：利用state创建许可数。

   public Semaphore(int permits) {

​    sync = new NonfairSync(permits);

   }

   NonfairSync(int permits) {

​     super(permits);

   }

   Sync(int permits) {

​     setState(permits);

   }

再看如何获取许可：

   public void acquire() throws InterruptedException {

​    sync.acquireSharedInterruptibly(1);

   }

   public final void acquireSharedInterruptibly(int arg)

​       throws InterruptedException {

​     if (Thread.interrupted())

​       throw new InterruptedException();

​     if (tryAcquireShared(arg) < 0)  //代表获取许可失败

​       doAcquireSharedInterruptibly(arg);

   }

  protected int tryAcquireShared(int acquires) {

​      return nonfairTryAcquireShared(acquires);

 }

 final int nonfairTryAcquireShared(int acquires) {

​       for (;;) {

​         int available = getState();

​         int remaining = available - acquires;

​         if (remaining < 0 ||

​           compareAndSetState(available, remaining)) //设置成功则获取许可成功

​           return remaining;

​       }

​     }

我们再看如何释放许可：

   public void release() {

​    sync.releaseShared(1);

   }

   public final boolean releaseShared(int arg) {

​     if (tryReleaseShared(arg)) { //释放许可成功，则走释放成功流程，唤醒其他等待线程

​       doReleaseShared();

​       return true;

​     }

​     return false;

   }

释放的许可直接加回state就行了

​     protected final boolean tryReleaseShared(int releases) {

​       for (;;) {

​         int current = getState();

​         int next = current + releases;

​         if (next < current) // overflow

​           throw new Error("Maximum permit count exceeded");

​         if (compareAndSetState(current, next))

​           return true;

​       }

​     }

# CountDownLatch原理

用来进行线程同步协作，等待所有线程完成倒计时。其中构造参数用来初始化等待计数值，await() 用来等待计数归零，countDown() 用来让计数减一：

public static void main(String[] args) throws InterruptedException {

​     CountDownLatch latch = new CountDownLatch(3);

​     new Thread(() -> {

​      log.debug("begin...");

​       sleep(1);

​      latch.countDown();

​      log.debug("end...{}", latch.getCount());

​     }).start();

​     new Thread(() -> {

​      log.debug("begin...");

​       sleep(2);

​      latch.countDown();

​      log.debug("end...{}", latch.getCount());

​     }).start();

​     new Thread(() -> {

​      log.debug("begin...");

​       sleep(1.5);

​      latch.countDown();

​      log.debug("end...{}", latch.getCount());

​     }).start();

​    log.debug("waiting...");

​    latch.await();

​    log.debug("wait end...");

}

先从构造看起：

   public CountDownLatch(int count) {

​     if (count < 0) throw new IllegalArgumentException("count < 0");

​     this.sync = new Sync(count);

   }

  Sync(int count) {

​     setState(count);

  }

再看await：

   public void await() throws InterruptedException {

​    sync.acquireSharedInterruptibly(1);

   }

   public final void acquireSharedInterruptibly(int arg)

​       throws InterruptedException {

​     if (Thread.interrupted())

​       throw new InterruptedException();

​     if (tryAcquireShared(arg) < 0)

​       doAcquireSharedInterruptibly(arg);

   }

  protected int tryAcquireShared(int acquires) {

​       return (getState() == 0) ? 1 : -1;  // 计数不为0，就进入等待队列

  }

再看countDown

   public void countDown() {

​    sync.releaseShared(1);

   }

   public final boolean releaseShared(int arg) {

​     if (tryReleaseShared(arg)) {

​       doReleaseShared(); //唤醒等待

​       return true;

​     }

​     return false;

   }

   protected boolean tryReleaseShared(int releases) {

​     // 递减计数； 过渡到零时就开始唤醒等待

​     for (;;) {

​         int c = getState();

​         if (c == 0)

​           return false;

​         int nextc = c-1;

​         if (compareAndSetState(c, nextc))

​           return nextc == 0;

​     }

   }
