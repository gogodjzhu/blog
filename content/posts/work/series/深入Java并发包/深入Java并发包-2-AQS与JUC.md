---
title: "深入Java并发包(2)-AQS与JUC"
date: 2019-11-12T22:45:21+08:00
description: "Java说要有并发包，所以有了AQS。作为并发包的基础，AQS提供了系列供子类实现的方法。将他们组合起来，便成了各种我们常用的锁/队列/同步器。本文以ReentrantLock和CountDownLatch为切入口扒一扒源码，讲一讲思路"
tags: ["JAVA", "并发编程"]
categories: "深入Java并发包"
draft: false
---

## AQS为何物？

AQS（`AbstractQueuedSynchronizer`）是JUC（`java.util.concurrent`）包提供的并发控制基础组件， 本质上就是将多个线程对竞争对象的操作变成有序的队列。 从类名也可以看出来，

- A (Abstract)指的是抽象性，是指导性的框架， 在本类中只会实现最基础的方法， 大多数的方法默认抛出异常`UnsupportedOperationException`， 子类可以按需重写自己需要的方法。
- Q (Queue)说明是本类通过显式队列来保存竞争线程的， 队列的节点是内部内`Node`， 双向队列。
- S (Synchronizer)即AQS的本质是一个同步器，或者可以简单理解它就是一个多用途的锁。

> **Synchronizer vs. synchronize**
>
> 当然地， ASQ内部绝对不会使用`synchronize`关键字， 因为AQS的出现本来就是为了更加轻量级地在java语言层面实现同步语义。 AQS依赖的主要是乐观锁技术， 而`synchronize`则是基于JVM底层对象头实现的悲观锁技术。悲观锁的特性导致`synchronize`在大多数竞争不激烈的场景性能不如AQS。

先看看AQS内部的全局变量只有三个:

```java
// 保存等待线程队列的头
private transient volatile Node head;
// 保存等待线程队列的尾
private transient volatile Node tail;
// 表示当前锁状态的属性， 但具体什么含义， 由子类确定
// 稍后聊到ReentrantLock和CountDownLatch的时候继续分析
private volatile int state

// 另外AQS还继承了AbstractOwnableSynchronizer，后者还有一个属性：
private transient Thread exclusiveOwnerThread; // 记录当前占有锁的线程
```

参与竞争的线程被抽象为`Node`对象，AQS将所有参与竞争的线程`Node`组织为队列。



### AQS#Node

作为AQS的内部类`Node`定义了几个`tryXXX()`系列方法。它们实际并没有做任何操作，仅仅抛出`UnsupportedOperationException`异常：

```java
protected boolean tryAcquire(int arg) {
    throw new UnsupportedOperationException();
}
```

类似`tryAcquire()`的方法还有`tryRelease`，`tryAcquireShared`，`tryReleaseShared`，`isHeldExclusively`共5个。这些方法围绕`state`实现锁的基本逻辑，子类拓展提供不同的功能例如：公平、非公平、同步等待、读写分离等。从这个角度来看，前面的的`tryAcquire()`做的事情是“获取锁”并不准确，实际做的是对`state`的竞争操作。



在介绍了AQS的基础框架之后，下面让我们来看一个依赖它实现的具体的常用子类。



## ReentrantLock

### 首次获取锁

`ReentrantLock`是常用的可重入锁。 它定义`state`为锁重入的次数， 0表示没有线程占有。 默认构造方法创建的是一个非公平的可重入锁。 也可以使用带参方法创建公平的重入锁， 区别在于创建的同步器`sync`是否公平。

```java
public ReentrantLock(boolean fair) {
    sync = fair ? new FairSync() : new NonfairSync();
}
```

ReentranLock调用`lock()`加锁方法最后落到`AQS#acquire(int)`上。此方法先尝试调用子类的`tryAcquire(int)`来做首次获取锁的尝试，成功则直接返回，失败则进入等待队列。

```java
// AQS.class
public final void acquire(int arg) {
    if (!tryAcquire(arg) && // 尝试一次获取锁
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg)) //一次失败入队列
        selfInterrupt();
}
```

前面介绍过，tryAcquire方法是AQS留给子类实现的同步逻辑，在ReentranLock中，`FairSync`和`NonfairSync`各自实现了此方法，基本原理都是通过`state`来判断锁定状态：当没有线程占有锁时`state=0`，任意线程获取锁都通过`CAS(0,1)`来争取锁，重入锁则`state+=1`；对应地，释放锁就是`state-=1`。

```java
// ReentrantLock$NonfairSync.class
// 非公平锁的tryAcquire逻辑
protected final boolean tryAcquire(int acquires) {
    // 即ReentrantLock$Sync#nonfairTryAcquire()
    return nonfairTryAcquire(acquires); 
}

// ReentrantLock$Sync.class
final boolean nonfairTryAcquire(int acquires) {
    // 注意此方法并未加锁，并发访问
    final Thread current = Thread.currentThread();
    // 尝试获取state值
    int c = getState();
    if (c == 0) {
        // state=0，进一步通过CAS安全修改
        if (compareAndSetState(0, acquires)) {
            // 获取成功，将占有锁的线程修改为当前
            setExclusiveOwnerThread(current);
            return true;
        }
    }
    else if (current == getExclusiveOwnerThread()) {
        // state!=0，目前占有锁的线程即是自己，重入锁
        int nextc = c + acquires;
        if (nextc < 0) // overflow
            // 重入次数int溢出
            throw new Error("Maximum lock count exceeded");
        setState(nextc); // 更新state状态为重入次数
        return true;
    }
    // 获取锁失败
    return false;
}

// ReentrantLock$FairSync.class
// 公平锁的tryAcquire逻辑
protected final boolean tryAcquire(int acquires) {
    //...
    int c = getState();
    if (c == 0) {
        // FairSync的关键在于CAS修改之前，先判断在等待锁队列中是否有前置节点
        if (!hasQueuedPredecessors() && 
            compareAndSetState(0, acquires)) {
            setExclusiveOwnerThread(current);
            return true;
        }
    }
    // ... 重入锁 和 获取锁失败 的处理跟NonfairSync完全一致
}
```

幸运的话，一次获取锁就能成功返回，这也是乐观锁最希望见到的情况。但如果一次获取锁失败，就需要进入队列等待锁，这时就会进入`AQS#acquireQueued(Node, int)`方法的逻辑。



### 等待锁

正如AQS类名的Q所指Queue，AQS提供的等待锁功能是通过队列来实现的，在进入`AQS#acquireQueued(Node, int)`方法前首先如要通过addWaiter(Node.EXCLUSIVE)方法构造一个Node节点。Node保存的信息包括:

- `volatile Node prev`节点保存前置节点
- `volatile Node next`节点保存后继节点
- `volatile Thread thread`节点保存当前线程
- `volatile int waitStatus`保存了当前节点的状态。 

```java
// AQS.java
final boolean acquireQueued(final Node node, int arg) {
    boolean failed = true; // 获取锁的结果标识
    try {
        boolean interrupted = false; // 在等待锁的过程中是否发生过中断
        // 死循环直到获取锁，或者异常退出。此循环是等待锁的核心逻辑
        for (;;) {
            final Node p = node.predecessor();
            // 前置节点就是队列头，认为大概率可以马上获取锁，则直接尝试获取锁
            if (p == head && tryAcquire(arg)) {
                // 获取锁成功，将队列头设为当前节点
                setHead(node);
                p.next = null; // help GC
                failed = false;
                return interrupted; // 返回等待过程中是否发生过中断
            }
            // parkAndCheckInterrupt方法触发线程park， 如果过程中发生中断返回
            // true， 本方法不可中断， 故仅标记interrupted交给外层方法处理
            if (shouldParkAfterFailedAcquire(p, node) &&
                    parkAndCheckInterrupt())
                interrupted = true;
        }
    } finally {
        if (failed) 
            // 异常时获取锁失败， 需要取消本次acquire请求
            // 并将此节点的状态修改为CANCELLED，结束锁争抢
            cancelAcquire(node);
    }
}

// AQS.class
// 在尝试获取锁失败之后，根据前置节点的状态来判断当前节点（即当前线程）是否需要进入park等待。
// 结合方法调用方可知，当此方法返回true时，会导致for循环暂停，直到unpark
private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
    int ws = pred.waitStatus;
    if (ws == Node.SIGNAL)
        // 前置节点的状态为SIGNAL，表示前置节点正常运行，后续节点需要等待前置节点
        // 的通知(SIGNAL)再做行动(是否进入park等待)
        return true;
    if (ws > 0) {
        // ws>0只有一个状态即CANCELLED，表示前置节点已经中止退出(退出前通过unpark唤醒
        // 了当前节点)那么就沿着队列找第一个未中止的节点为当前的新前置节点
        do {
            node.prev = pred = pred.prev;
        } while (pred.waitStatus > 0);
        pred.next = node;
    } else {
        // 在park之前将前置节点的状态修改为SIGNAL，提醒它记得来唤醒(unpark)当前节点
        compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
    }
    return false;
}

// AQS.class
// 在判断当前节点需要进入park等待时，就掉用此方法
// 注意返回是否异常的标志，这作为可中断lock的依据
private final boolean parkAndCheckInterrupt() {
    // park等待，阻塞方法
    // 线程异常 or 其他线程调用unpark方才退出阻塞
    LockSupport.park(this);
    // 线程异常导致的park退出阻塞，此方法会返回true
    // 反之，其他线程释放锁唤醒当前线程时，此方法返回false
    return Thread.interrupted();
}
```

> **CLH vs. Sync(CAS)**
>
> 对比CLH和Sync的自旋判断可以发现， 前者判断的是变量`pred.locked`的值， 只有前置节点会修改它的状态， 不存在线程竞争。后者判断的是变量`AQS.state`， 它可以通过CAS被所有竞争线程并发安全地修改。 同样的逻辑使用CAS实现比CLH的经典实现更为优雅， 而且可以减少bug(参考深入Java并发包(1) – 什么是锁?#死锁问题)。 这就是底层技术进步带来的优势。



可以看到不同Sync底层均调用AQS的acquire方法实现类似CLH的自旋判断， 

> 可中断 vs. 不可中断
>
> 发现acquire方法在排队得到锁之后， 还根据中断标志选择调用了`selfInterrupt()`方法中断自己。 这是因为`acquire()->acquireQueued()->parkAndCheckInterrupt()->LockSupport.part()`挂起线程等待唤醒(相当于CLH中while自旋)。 但唤醒分两种情况:
>
> 1. 前置节点unpark当前线程， 此时不会发生中断， 也不调用`selfInterrupt()`方法
>
> 2. 其余线程中断当前线程， 导致`LockSupport.part()`异常唤醒(不同于`Thread.sleep()`或者`wait()`， park方法被中断之后只会唤醒， 不主动抛出InterruptException， 只能通过`Thread.interrupted()`判断唤醒是否由中断导致)， 而`parkAndCheckInterrupt`被中断唤醒之后还通过`Thread.interrupted()`清空了中断状态并返回`true`。 外部方法通过`parkAndCheckInterrupt`返回是否`true`判断中断是否发生。 此时`interrupted`变量会记下此状态， 等到成功获取锁之后再中断自己。 

来到这里， 正常等待锁的线程都在park状态了， 直到当前持有锁的线程调用unlock()方法释放锁:

```java
// ReentrantLock.java
public void unlock() {
    sync.release(1);
}
// AQS.java
public final boolean release(int arg) {
    if (tryRelease(arg)) {
        Node h = head;
        if (h != null && h.waitStatus != 0)
            unparkSuccessor(h);
        return true;
    }
    return false;
}
```

有了前面加锁的铺垫， 释放锁差不多就是一个逆运算。 直接调用AQS的`release()`方法， 在`tryRelease()`中递减state， 在`unparkSuccessor()`中unpark`head`的`successor`，即头结点的下一个节点。 

注意这里`unparkSuccessor`方法unpark的只是head的下一个节点， 不论公平与非公平均调用此方法释放锁， 导致的一个问题就是: **事实上非公平锁的”非公平”只体现在入队列前的第一次抢夺锁， 一旦抢夺失败进入到队列， 一样是FIFO， 非公平又变得公平了。**

至此，ReentrantLock的简单加锁流程就完整了。另外ReentrantLock还提供了一些更丰富的功能。例如：

- `ReentrantLock#lockInterruptibly()`
  实现可中断的锁。实现方法是，通过park循环等待中，parkAndCheckInterrupt捕捉到异常即马上抛出异常结束等待。
- `ReentrantLock#tryLock()`
  只尝试一次获取锁，失败立马返回。实现方法是，取消park循环等待。
- `ReentrantLock#tryLock(long， TimeUnit)`
  带超时时间的获取锁。实现方法是，底层将`LockSupport.park(Object)`改为`LockSupport.parkNanos(Object, long)`，可以实现在等待指定时间无唤醒之后主动从park状态醒来。

## CountDownLatch

Latch原意为门栓， 描述CountDownLatch的作用很贴切， 就是让在门外等待(`wait()`方法)的线程等在门外。初始化CountDownLatch的时候指定state的值(>=0)，每次调用`countDown()`均会递减AQS的`state`， 直到`state`降为0打开门栓， 所有等待线程同时得到许可继续运行。

不同于ReentrantLock， Latch同时给多个线程授予许可， 所以ReentrantLock调用的`AQS#acquire`方法的阻塞排他特性不再适用， `CountDownLatch#wait()`调用`AQS#acquireSharedInterruptibly`方法， 它是可中断的， 调用的子类方法也变成的`tryAcquireShared`:

```java
// CountDownLatch.class
public void await() throws InterruptedException {
    sync.acquireSharedInterruptibly(1);
}

// AQS.class
public final void acquireSharedInterruptibly(int arg)
        throws InterruptedException {
    if (Thread.interrupted())
        throw new InterruptedException();
    if (tryAcquireShared(arg) < 0)
        doAcquireSharedInterruptibly(arg);
}

// CountDownLatch.class
protected int tryAcquireShared(int acquires) {
    return (getState() == 0) ? 1 : -1;
}

// AQS.java
private void doAcquireSharedInterruptibly(int arg)
        throws InterruptedException {
    final Node node = addWaiter(Node.SHARED);
    boolean failed = true;
    try {
        for (;;) {
            final Node p = node.predecessor();
            if (p == head) {
                // 调用CountDownLatch方法， 返回>0表示获取锁成功
                int r = tryAcquireShared(arg);
                if (r >= 0) {
                    // ReentrantLock仅仅将当前节点设为头， 而Latch还需要设置
                    // Propagate保证解锁状态传递下去
                    setHeadAndPropagate(node, r);
                    p。next = null; // help GC
                    failed = false;
                    return;
                }
            }
            if (shouldParkAfterFailedAcquire(p, node) &&
                    parkAndCheckInterrupt())
                throw new InterruptedException();
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}
```

注意Latch在释放锁之后， 头结点还需要设置传递， 将解锁状态沿着队列传递下去使所有线程均被唤醒。

```java
// AQS
private void setHeadAndPropagate(Node node, int propagate) {
    Node h = head; // Record old head for check below
    setHead(node);
    if (propagate > 0 || h == null || h.waitStatus < 0 ||
            (h = head) == null || h.waitStatus < 0) {
        Node s = node.next;
        if (s == null || s.isShared())
            doReleaseShared();
    }
}

// AQS
private void doReleaseShared() {
    for (;;) {
        Node h = head;
        if (h != null && h != tail) {
            int ws = h.waitStatus;
            if (ws == Node.SIGNAL) {
                if (!compareAndSetWaitStatus(h, Node.SIGNAL, 0))
                    continue; // loop to recheck cases
                unparkSuccessor(h);
            }
            else if (ws == 0 &&
                    !compareAndSetWaitStatus(h, 0, Node.PROPAGATE))
                continue; // loop on failed CAS
        }
        if (h == head) // loop if head changed
            break;
    }
}
```



# 总结

AQS作为框架依靠CAS技术提供了线程安全地修改`status`共享变量的前提，又通过队列记录了参与修改`status`的线程信息，而`LockSupport`类则使得高效的线程等待成为可能。

基于AQS的这些特性，子类实现获取/释放，公平/非公平锁， 可中断/不可中断等功能。

以上把Java并发的基础AQS和依赖它实现的`Lock`和`Latch`都分析了一遍， 下一篇我们看看开发中常用的并发容器， 如何依赖这些工具并发安全地读写数据。