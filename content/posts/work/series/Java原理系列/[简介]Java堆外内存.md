---
title: "[简介]Java堆外内存"
date: 2020-02-05T04:08:00+08:00
description: "简单介绍了堆外内存的一些基础特性和基础使用方法"
featured_image: "http://minio.gogodjzhu.com/images/20210403_120453_416044f6-3b96-4d81-a7ab-bbc056e39398.png"
images: ["http://minio.gogodjzhu.com/images/20210403_120453_416044f6-3b96-4d81-a7ab-bbc056e39398.png"]
tags: ["JAVA"]
categories: "Java原理系列"
draft: false
---

堆外内存是不受GC控制的内存空间，相对来说灵活度更高。大部分Java开发不会直接用到堆外内存，但对一些框架应用(如`Kafka`, `Netty`)堆外内存是必须牢牢掌控的一份宝藏，因为它最起码具有以下这些好处：

1. 空间不受堆大小限制(但可通过-XX:MaxDirectMemorySize参数控制)
2. 可以自定义内存分配和回收策略，不受JVM gc约束
3. 可以使用`零拷贝`等高级特性，这在网络IO是极大的优势



# 构造方法

DirectByteBuffer的构造方法如下，

```java
DirectByteBuffer(int cap) {
    // 按照ByteBuffer的抽象定义初始化对应的属性, 这些属性是跟DirectBuffer
    // 的实现无关
    super(-1, 0, cap, cap); // mark, position, limit, capacity
    // 根据是否页对齐 和 申请内存大小cap 来判断堆外内存是否足够分配
    boolean pa = VM.isDirectMemoryPageAligned();
    int ps = Bits.pageSize();
    // 最少创建size=1
    long size = Math.max(1L, (long)cap + (pa ? ps : 0));
    Bits.reserveMemory(size, cap); // 核心方法1. 检查堆外内存空间

    long base = 0;
    try {
        base = unsafe.allocateMemory(size); // 核心方法2. 分配堆外内存空间,返回内存地址
    } catch (OutOfMemoryError x) {
        Bits.unreserveMemory(size, cap);
        throw x;
    }
    // 初始化内存
    unsafe.setMemory(base, size, (byte) 0);
    // 计算得到Buffer使用的内存地址，这一套算法看不太懂
    if (pa && (base % ps != 0)) {
        // Round up to page boundary
        address = base + ps - (base & (ps - 1));
    } else {
        address = base;
    }
    // 核心方法3. 构造Cleaner，用于回收堆外内存
    cleaner = Cleaner.create(this, new Deallocator(base, size, cap));
    att = null;
}
```

JVM的可用堆外内存空间跟实际的堆外内存空间(物理内存空间)不同。逻辑上由Bits来统计当前JVM的堆外内存使用情况，而实际分配物理内存的工作留给`unsafe`。



## 逻辑空间申请

首先通过`Bits.reserveMemory(size, cap)`方法判断逻辑上的堆外空间是否足够，不足时会休眠等待重试(设有超时时间约0.5秒)。

```java
static void reserveMemory(long size, int cap) {

    // 从VM读取当前JVM可分配的最大堆外内存, 这个值可以通过'-XX:MaxDirectMemorySize=<size>'配置
    if (!memoryLimitSet && VM.isBooted()) {
        maxMemory = VM.maxDirectMemory();
        memoryLimitSet = true; // maxMemory在启动时确定, JVM运行过程中不会改变
    }

    // 先直接判断是否满足
    if (tryReserveMemory(size, cap)) {
        return;
    }

    final JavaLangRefAccess jlra = SharedSecrets.getJavaLangRefAccess();
    // 通过jlra可以主动触发其他堆外内存Cleaner的回收方法, 返回true表示有成功回收堆外内存
    while (jlra.tryHandlePendingReference()) {
        // 回收堆外内存后重新判断
        if (tryReserveMemory(size, cap)) {
            return;
        }
    }

    // 触发系统gc
    System.gc();

    boolean interrupted = false;
    try {
        long sleepTime = 1;
        int sleeps = 0;
        // 循环等待堆外内存空间是否足够, 每次等待的时间翻倍, MAX_SLEEPS=9
        while (true) {
            if (tryReserveMemory(size, cap)) {
                return;
            }
            if (sleeps >= MAX_SLEEPS) {
                break;
            }
            if (!jlra.tryHandlePendingReference()) {
                try {
                    Thread.sleep(sleepTime);
                    sleepTime <<= 1;
                    sleeps++;
                } catch (InterruptedException e) {
                    interrupted = true;
                }
            }
        }

        // 仍然无法分配足够的堆外内存, 抛出异常
        throw new OutOfMemoryError("Direct buffer memory");

    } finally {
        if (interrupted) {
            // don't swallow interrupts
            Thread.currentThread().interrupt();
        }
    }
}

private static boolean tryReserveMemory(long size, int cap) {
    /**
     * maxMemory: 从VM读取当前JVM可分配的最大堆外内存, 这个值可以通过'-XX:MaxDirectMemorySize=<size>'配置
     * totalCapacity:AtomicLong : 当前JVM已经分配出去的堆外内存大小
    **/

    long totalCap;

    // cap < maxMemory - totalCap, 未超过JVM允许的堆外内存最大值. 
    while (cap <= maxMemory - (totalCap = totalCapacity.get())) {
        // 改变内存容量
        if (totalCapacity.compareAndSet(totalCap, totalCap + cap)) {
            reservedMemory.addAndGet(size);
            count.incrementAndGet();
            return true;
        }
    }
    return false;
}
```



## 物理空间申请

在确定逻辑上可以分配足够多的堆外内存空间之后，使用`unsafe`工具类真正地去申请物理堆外内存。如果unsafe无法分配足够多的内存空间时(OOM)需要通过Bits回收逻辑上分配出去的内存空间。

```java
long base = 0;
try {
    base = unsafe.allocateMemory(size);
} catch (OutOfMemoryError x) {
    Bits.unreserveMemory(size, cap);
    throw x;
}
```



## 创建堆外内存回收器Cleaner

由于堆外内存不受gc控制，所以需要在堆外内存使用完毕后主动释放。不同于**数据库连接**这样的堆外资源常用的通过`try-finally`代码块显示释放资源的做法，`DirectByteBuffer`采用`sun.misc.Cleaner`来实现自动的资源回收。

`Cleaner`继承了`PhantomReference`(虚引用)，虚引用类似于`finalize()`方法的升级版，可以在垃圾回收器回收虚引用所指的对象(即DirectByteBuffer对象)之前，通过`引用队列`主动执行一些资源释放的操作（在这里就是执行`Deallocator#run()`方法）。关于虚引用的细节可以参考[这篇文章](https://www.gogodjzhu.com/202002/java-garbage-collector-and-reference-objects/)。

```java
private static class Deallocator implements Runnable {

    private static Unsafe unsafe = Unsafe.getUnsafe();

    private long address;
    private long size;
    private int capacity;

    private Deallocator(long address, long size, int capacity) {
        assert (address != 0);
        this.address = address;
        this.size = size;
        this.capacity = capacity;
    }

    // 在回收DirectByteBuffer对象前会回调此方法
    public void run() {
        if (address == 0) {
            // Paranoia
            return;
        }
        // 释放物理堆外内存
        unsafe.freeMemory(address);
        address = 0;
        // 释放逻辑堆外内存
        Bits.unreserveMemory(size, capacity);
    }

}
```

