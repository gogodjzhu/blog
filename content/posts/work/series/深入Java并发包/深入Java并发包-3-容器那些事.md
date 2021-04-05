---
title: "深入Java并发包(3)-容器那些事"
date: 2019-11-12T22:55:21+08:00
description: "说并发包，不能不说并发安全的容器。本章选择几个最常用的容器，结合系列文章分析过的基础加锁组件，看看JUC并发容器底层原理。着重分析线程安全部分。"
tags: ["JAVA", "并发编程"]
categories: "深入Java并发包"
draft: false
---

## CopyOnWriteArrayList vs. ArrayList

CopyOnWriteArrayList 是 ArrayList的线程安全版本， 底层保存均用数组实现， 并含有一个全局变量ReentrantLock实现线程安全。 

```java
// CopyOnWriteArrayList.java
final transient ReentrantLock lock = new ReentrantLock();
private transient volatile Object[] array; // 注意使用了volatile保证可见性
```

在`add`， `remove`， `clear`， `set`这些会改变数据的方法上通过`ReentrantLock.lock()`方法加锁， 并且使用复制数组的方式改变数组长度(区别于ArrayList通过`直接操作数组`+`按1.5比例扩容`的方式操作数组)。 而get方法不加锁。 

```java
// 尾部追加元素
public boolean add(E e) {
    final ReentrantLock lock = this.lock;
    lock.lock(); // 典型的lock + finally{unlock}操作方式
    try {
        Object[] elements = getArray();
        int len = elements.length;
        // copy数组， modify数组， set数组
        Object[] newElements = Arrays.copyOf(elements, len + 1);
        newElements[len] = e;
        setArray(newElements);
        return true;
    } finally {
        lock.unlock();
    }
}

// 由于使用copy+modify+set方法修改数组， 所以get的时候不需要加锁
public E get(int index) {
    return get(getArray(), index);
}
```

## ConcurrentHashMap vs. HashMap vs. HashTable

HashMap的基本原理是通过计算每一个元素的key的hash值， 将其映射到数组中保存。 当出现hash冲突时， 退化为链表(JDK1.8优化为红黑树)保存。 

对标非线程安全的HashMap， JDK1.0开始提供了一个HashTable， 它提供的线程安全方法比较粗暴， 把所有`get`， `put`， `clear`， `size`等方法加上`synchronized`关键字。 等于所有操作均会锁住整个map， 性能较低。

JDK1.5开始， 跟随juc包一起发布了高性能并发安全的`ConcurrentHashMap`。 在JDK1.7和JDK1.8中实现有些微的区别。 下面分别介绍:

### ConcurrentHashMap@JDK1.7

`ConcurrentHashMap`基于桶保存数据， 它的高性能主要来自于三方面:

1. 分桶锁， 不再锁全表
2. 利用`ReentrantLock`乐观锁， 增加加锁性能
3. 读操作不加锁， 通过`Unsafe.getObjectVolatile`实现volatile语义， 确保读取最新数据

ConcurrentHashMap的分桶数量在创建map对象的时候一经确定， 就不再修改， 所以第一步要确定的中间就是如何计算分桶的数量。 所有的构造方法最终调用的都是`ConcurrentHashMap(int initialCapacity， float loadFactor， int concurrencyLevel)`。 `initialCapacity`和`loadFactor`分别指定map初始化的容量和分桶扩容阈值。 `concurrencyLevel`指定预期的并发数， 最终的分桶数为满足大于`concurrencyLevel`的最小的2的n次幂， 默认为16。 

注意到构造方法中仅创建了分桶数组`segments`和第一个分桶， 其余的分桶只有在需要写入数据时(具体说就是`put`方法)才延后创建。 

```java
// ConcurrentHashMap
public V put(K key, V value) {
    Segment<K,V> s;
    if (value == null)
        throw new NullPointerException();
    // 取hash值的高位按位与上segmentMask得到分桶的在数组内的相对位置j
    int hash = hash(key);
    int j = (hash >>> segmentShift) & segmentMask;
    // 因为分桶数量固定， 桶一旦实例化就不会改变， 所以这里使用非volatile的方法
    // 获取桶， 只有发现为null时， 才在ensureSegment内部通过CAS创建新桶
    if ((s = (Segment<K，V>)UNSAFE.getObject(segments, (j << SSHIFT) + SBASE)) == null)
        s = ensureSegment(j);
    // 真正的写入逻辑落到Segment的put方法
    return s.put(key, hash, value, false);
}
```

分桶类`Segment`继承自`ReentrantLock`， 在put写入数据时首先尝试调用`tryLock`获取锁， 成功的时候执行写入， 跟HashMap(JDK1.7)的写入一模一样。 如果获取失败， 则先尝试若干次`tryLock`， 还不成功， 则调用阻塞`lock()`方法。 

```java
// Segment extends ReentrantLock
// onlyIfAbsent 为true则只替换已有key，已存在不会覆盖仅返回旧值
final V put(K key, int hash, V value, boolean onlyIfAbsent) {
    // 尝试获取锁， 失败继续在scanAndLockForPut等待(同时做了很有意思的warm up)
    // 工作， 具体看下文
    HashEntry<K,V> node = tryLock() ? null :
            scanAndLockForPut(key, hash, value);
    V oldValue;
    try {
        HashEntry<K,V>[] tab = table;
        // Segment内部还是用了同样的hash值寻址Entry
        int index = (tab.length - 1) & hash;
        HashEntry<K,V> first = entryAt(tab,index);
        for (HashEntry<K,V> e = first;;) {
            // 目标已成链， 则通过e。next寻找key相等的Entry
            if (e != null) {
                K k;
                if ((k = e.key) == key ||
                        (e.hash == hash && key.equals(k))) {
                    oldValue = e.value;
                    if (!onlyIfAbsent) {
                        e.value = value;
                        ++modCount;
                    }
                    break;
                }
                e = e.next;
            }
            else {
                if (node != null)
                    node.setNext(first);
                else
                    node = new HashEntry<K,V>(hash, key, value, first);
                int c = count + 1;
                // 数量大于阈值， 且还没到达整个链表数量的最大值， 将其rehash， 
                // 创建更多的链表， 摊平数据
                if (c > threshold && tab.length < MAXIMUM_CAPACITY)
                    // rehash方法每次扩容一倍， 链表长度永远为2的n次幂
                    rehash(node);
                else
                    setEntryAt(tab, index, node);
                ++modCount;
                count = c;
                oldValue = null;
                break;
            }
        }
    } finally {
        unlock();
    }
    return oldValue;
}

// tryLock尝试获取锁n次， 再lock阻塞锁
private HashEntry<K,V> scanAndLockForPut(K key, int hash, V value) {
    HashEntry<K,V> first = entryForHash(this, hash);
    HashEntry<K,V> e = first;
    HashEntry<K,V> node = null;
    int retries = -1;
    while (!tryLock()) {
        HashEntry<K,V> f; // to recheck first below
        if (retries < 0) {
            if (e == null) {
                if (node == null) // speculatively create node
                    node = new HashEntry<K,V>(hash, key, value, null);
                retries = 0; 
            }
            else if (key.equals(e.key))
                retries = 0;
            else
                e = e.next;
        }
        else if (++retries > MAX_SCAN_RETRIES) {
            // 超过最大重试次数， 阻塞锁
            lock();
            break;
        }
        else if ((retries & 1) == 0 &&
                (f = entryForHash(this, hash)) != first) {
            e = first = f; // re-traverse if entry changed
            retries = -1;
        }
    }
    return node;
}
```

> **scanAndLockForPut warm up?**
>
> 有趣的地方在于scanAndLockForPut不仅做了尝试获取锁的工作， 还在tryLock的过程中遍历了hash值对应的链表(如果已经key冲突)， 判断是否有相同的key的数据。 如果已有， 返回null; 如果没有， 返回node。 上层方法获取锁之后可以直接利用此结果。
> 这样当前线程获取锁的时候就不需要一边占用锁， 一边耗时去做这种遍历判断。 当然， 在最后一个else if中还判断了链表是否在这个过程中修改了值， 如果修改了， 那就将整个过程重新来一次。 性能消耗? 问题不大， 因为当前线程tryLock等着也是等着。 在doc中， 称这种方法叫热身? warm up。
>
> 感兴趣的朋友可以看看[这里](https://stackoverflow。com/questions/31704209/scanandlockforput-in-concurrenthashmap-jdk1-7)， 还有作者[Doug亲自回复的mail ](http://altair。cs。oswego。edu/pipermail/concurrency-interest/2014-August/012881。html)

相对于put操作， `get(Object key)`方法就简单多了， 通过两个`Unsafe.getObjectVolatile`方法先后获取Segment和Entry即可。 这里不再赘述。

### ConcurrentHashMap@JDK1.8

JDK7对Segment的实现跟HashMap几乎一致，所以HashMap存在的hashCode大量冲突导致的链表读性能差的问题也被一并带入。JDK8对此进行了改进(连ConcurrentHashMap和HashMap一并改进)，当链表的长度大于阈值(默认为8)时，会将链表变成红黑树，从而实现更快的读取。