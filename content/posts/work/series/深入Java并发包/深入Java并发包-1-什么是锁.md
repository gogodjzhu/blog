---
title: "深入Java并发包(1)-什么是锁?"
date: 2019-11-12T21:45:21+08:00
description: "加锁是实现有序并发控制的常见方法，本文先介绍了锁的核心概念及其实现"
tags: ["JAVA", "并发编程"]
categories: "深入Java并发包"
draft: false
---

锁的设计目的是为了让多线程并发时，控制线程对某一资源的使用。简单的情况就是控制同时只能有一个持有锁的线程访问资源，这是本文要讨论的。



## 锁的基础

讨论锁， 必然会涉及3种对象:

1. 被锁住的”对象”（称为临界区）
2. 锁本身
3. 要访问被锁”对象”的实例（即竞争线程）

锁的基础是可以绝对可靠的排他， 否则开锁的一瞬间多个竞争线程同时进入临界区， 这就违背了锁的语义。

先看一段代码:

```java
public class BadLock {
    private static boolean lock = false;
    public void lock(boolean lock) {
        while (lock);
        lock = true;
    }

    public void unlock(boolean lock) {
        lock = false;
    }
}
```

`lock()`方法通过while一直循环判断`lock`是否为`true`(这个动作称为自旋spin)，`lock`为`true`表示有其他线程占用资源，需要当前线程等待。 
直到占有锁的线程通过`unlock()`方法释放锁，即将`lock`置为`false`，当前线程发现`lock==false`跳出自旋，获取锁。 

看似简单，但是事实却比这复杂。加锁动作可以简化为三个核心步骤：

1. `lock`被持有锁的线程置为`false`
2. 当前线程发现`lock`变为`false`
3. 当前线程将`lock`变为`true`（开始进行排他操作）

操作之中，可能有更多的更多的其余线程也在同时改变`lock`的状态。

- 如果步骤1-2之间`lock`被其他线程改为`true`， 当前线程就无法发现`lock`的改变， 自旋继续。 这个没有问题。

- 如果步骤2-3之间`lock`被其他线程改为`true`(也是通过`lock()`方法)， 那就会有多个线程同时认为自己获取锁， 进入到临界区。

这里的主要问题就在于步骤2-3的操作不是原子性的，可能被插入其他的操作。好在现代的操作系统中提供了read-modify-write（RMW）原子操作，可以原子地执行读-改-写操作。让2-3变成原子操作， 最简单的锁(SimpleLock)就实现了。这种锁简单，但是不够公平(*从先申请先获取的角度来说公平*)，因为每次lock释放之后，所有等待锁的线程不论先后有同等机会获取锁。 



## CLH自旋锁

CLH(Craig，Landin and Hagersten。名称来自发明人的名字 )自旋锁是一种基于隐式链表（节点里面没有next指针）的公平的自旋锁。 它的原理是**通过一个prev变量将所有等待锁线程排队， 任意一个线程只等待队列中前一个线程释放锁**，下图解释了自选锁的工作：

![img](http://minio.gogodjzhu.com/images/20210403_222838_7e4e3dd5-e276-4b71-9fb5-bad5c2bfcc24.png)



#### 死锁问题

注意这里在设计上将当前节点`myNode`和前驱节点`myPred`区分开， 是否想过可以将`myPred`去掉， 简化成只有`myNode`? 答案是不行， 因为仅有前驱节点的时候， 如果不加其余优化， 会导致死锁。 比方说现在有T1和T2两个线程， T1是T2的前驱节点且当前拥有锁。 T1释放锁(position T1-1)之后马上又尝试获取锁(position T1-3)， 在这个过程中T2将前驱T1写入tail(position T2-1)， 之后T1获取前驱节点信息(position T1-2)是其本身， 这时T1并未获取锁， 所以永远在等待自身释放锁(position T1-3)。
造成这种现象的根源在于tail属于共享对象， 对它的切换不是原子性的， 以至于使得前驱节点指向当前节点。 那么我们通过线程变量pred， 解开这种循环依赖， 搞定。

```java
public class CLHLock implements Lock {
    private final AtomicReference<QNode> tail;
    private final ThreadLocal<QNode> myPred;
    private final ThreadLocal<QNode> myNode;
    //。。。 省略初始化操作

    public void lock() {
        QNode qnode = myNode。get();
        qnode.locked = true;
        QNode pred = tail.getAndSet(qnode);// position T2-1， position T1-2
        myPred.set(pred);
        while (pred.locked){} // position T2-2， position T1-3
    }

    public void unlock() {
        QNode qnode = myNode。get();
        qnode.locked = false;
        myNode.set(myPred.get()); // position T1-1
    }

    private static class QNode { volatile boolean locked;}
}
```



## MCS自旋锁

设计随着环境变化， CLH如果遇上NUMA（Non Uniform Memory Access，非统一内存访问，是一种用于多处理器的电脑内存体设计，内存访问时间取决于处理器的内存位置。 在NUMA下，处理器访问它自己的本地存储器的速度比非本地存储器快一些。）架构的系统， 也出现了新的优化空间。 由于在传统的架构当中， 多个线程共享L1 cache， CLH中`while`命令自旋检查的对象虽然属于不同的线程， 但是同样落在一个cache中。 而面对NUMA这种架构， 不同线程的变量可能在不同的cache中， 这样就导致了需要加一层cache一致性的消耗。 所以MCS应运而生。

MCS是一种基于显式链表(节点里面拥有next指针)公平的自旋锁。 它的原理是让当前节点保存一个指向后继节点的指针， 并令所有的后继节点均在自己的对象上自旋， 持有锁的节点释放锁的时候主动通知后继节点。 

注意， MCS的主要优势在于`自旋的对象`是线程内对象还是跨线程对象， 因为这两种资源的访问在NUMA架构下可能会导致比较大的性能差异。

![img](http://minio.gogodjzhu.com/images/20210403_224125_51150a63-4005-4402-b868-efc57d9f88b3.png)

## AQS 的CLH增强锁

AbstractQueuedSynchronizer(AQS)是Java并发包的基石之一，`ReentrantLock`， `ReentrantReadWriteLock`， `CountDownLatch`， `Semaphore`等类均继承自此抽象类， 按照AQS的文档说法， 此类提供了一个框架， 用以实现依赖同步队列。 。
AQS的核心在于以下几个部分:

- Node 等待节点。 类似于CLH中的QNode， 用以实现等待线程的双向队列。
- state 的管理， 这部分决定了不同AQS实现类的不同特性。 AQS通过暴露`tryAcquire`和`tryRelease`等几个待实现方法的方式给子类来管理state。
- 基于LockSupport的竞争锁机制。 这部分是AQS的核心。 

系列文章下一节我们将会以ReentrantLock， CountDownLatch为例， 深入了解如何借助AQS实现不同类型的锁。