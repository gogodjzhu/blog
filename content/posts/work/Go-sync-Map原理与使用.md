---
title: "Go sync.Map原理&使用"
date: 2018-12-23T14:28:20+08:00
tags: ["golang"]
draft: false
---

Go1.9中新增了一个新的类型`sync.Map`，它提供并发`map`的功能，本文针对其讨论两个问题：

1. 在现有标准库下，为什么需要添加sync.Map
2. sync.Map的原理

为什么要有sync.Map？在没有sync.Map的日子里，我们可以通过sync.Mutex或者sync.RWMutex来实现线程安全的Map，把读写操作变成单线程操作来实现。下面是我们使用sync.RWMutex实现的一个简单的线程安全Map。

```go
package RegularIntMap

type RegularIntMap struct {
    sync.RWMutex
    internal map[int]int
}

func NewRegularIntMap() *RegularIntMap {
    return &RegularIntMap{
        internal: make(map[int]int),
    }
}

func (rm *RegularIntMap) Load(key int) (value int, ok bool) {
    rm.RLock()//读锁，可并发
    result, ok := rm.internal[key]
    rm.RUnlock()
    return result, ok
}

func (rm *RegularIntMap) Delete(key int) {
    rm.Lock()
    delete(rm.internal, key)
    rm.Unlock()
}

func (rm *RegularIntMap) Store(key, value int) {
    rm.Lock()
    rm.internal[key] = value
    rm.Unlock()
}
```

但哪怕使用了读写分离锁，当代码运行在多核CPU下(通常情况下是超过8/16核以上的服务器)性能依然堪忧。因为以下几个原因：

1. `reflect.New`很慢(map类型安全的底层是通过反射机制来实现的)
2. `sync.RWMutex`很慢
3. `atomic.AddUint32`很慢
4. 所有的cpu都在读写同一个内存地址

为了解决这些问题，golang1.9推出了sync.Map包以支持更高性能读写操作。简单使用如下：

```go
type mapInterface interface {
    //类似于java的Map.get()，返回命中key的value，并带有一个bool表示是否命中
    Load(interface{}) (interface{}, bool)

    //类似于java的Map.set()，写入key-value
    Store(key, value interface{})

    //尝试从map中拉取key-value，如果不存在，则写入key-value。loaded表示是否load操作命中
    LoadOrStore(key, value interface{}) (actual interface{}, loaded bool)

    //删除一个key-value
    Delete(interface{})

    //遍历所有的key-value调用func，func应该返回一个bool值。func返回false会导致遍历终止
    Range(func(key, value interface{}) (shouldContinue bool))
}

func syncMapUsage() {
    fmt.Println("sync.Map test (Go 1.9+ only)")
    fmt.Println("----------------------------")

    // Create the threadsafe map.
    var sm sync.Map

    // Fetch an item that doesn't exist yet.
    result, ok := sm.Load("hello")
    if ok {
        fmt.Println(result.(string))//注意这里，sync.Map容器是以interface{}的方式保存对象的，所以需要进行类型转换。也就是说sync.Map是类型不安全的
    } else {
        fmt.Println("value not found for key: `hello`")
    }

    // Store an item in the map.
    sm.Store("hello", "world")
    fmt.Println("added value: `world` for key: `hello`")

    // Fetch the item we just stored.
    result, ok = sm.Load("hello")
    if ok {
        fmt.Printf("result: `%s` found for key: `hello`\n", result.(string))
    }
}
```

总结一下，`sync.Map`像是一个不够完善的容器，比起已有的map主要存在以下不足：

1. 低并发情况下的性能不足
2. 冗余数据(两个不同的map，后面会谈到)
3. 缺少类型安全控制
4. 有限的api。如不支持`len`操作

在[这篇文章](https://medium.com/@deckarep/the-new-kid-in-town-gos-sync-map-de24a6bf7c2c)中，作者对sync.Map和RWMutex实现的Map做了性能比较，可以看出在核数超过4的时候，sync.Map的性能才超过RW锁。

- `sync.Map` vs. `RWMutex`

  ![img](http://minio.gogodjzhu.com/images/20210405_121039_1804e52d-8137-4085-819b-1db20f8e6841.png)



那么在代码底层，sync.Map是如何实现高并发场景下的性能呢？答案是两个独立的map：

- 一个通过`atomic.Value`保存的`read-only Map(read)`；

- 一个使用`sync.mutex`控制的`read-write Map(dirty)`。

而两个map的value保存的都不是实际的对象，而是指相同对象的指针，所以通过两个map去读写的都是相同的对象。下面看一下sync.Map的内部结构图，

- sync.Map读操作`load(key)`示意图（相同颜色的箭头组成一条操作路径）：

![img](http://minio.gogodjzhu.com/images/20210405_121035_7d2ba2a9-2918-4bb0-9f3f-a2062eb81016.png)

可以看到当我们load一个key的时候，会首先到read中进行检索(对read的读操作并不加锁，所以并发性能较高)，如果找不到才尝试去dirty中寻找，而dirty的读操作是加锁的，防止此时有新的key添加或者删除，导致dirty状态不一致。而每一次未命中read却命中dirty的操作都会使得一个misses计数器加一，当这个值超过dirty的长度时，就会触发一次数据从dirty到read的完整迁移。由于迁移只是把dirty内部保存指针的map复制替换到read中，比起迁移对象本身还是要快很多。

- sync.Map写入数据`store(key)`操作示意图

  ![img](http://minio.gogodjzhu.com/images/20210405_121255_58e0ce59-0ecc-486d-b942-31bd2f2242af.png)

有了前面对load操作的了解，store操作就可以触类旁通了。因为read和dirty内部map保存的都是指向实际保存对象的指针(通过entry进行封装)，所以不论是通过read还是dirty操作的都是相同的对象实体。这在更新操作的时候变得非常有效率。如上图所示，让我们往一个在read中存在的键写入新值时，直接拿到entry，对其指向的内存空间进行CAS操作即可，不用加锁。只有在写入read中不存在的键时，才会通过dirty加锁操作写入新的数据。 

最后是删除操作，分两种情况：

1. 删除的key仅在dirty中存在。此时只需要简单的将key从dirty内部的map中删除即可。
2. 删除的key在read中存在。这种情况下会先把read内部map中对应key的值设为expunged(一个指针标记)，但不会对dirty做任何操作。此时因为read中对应的key依然存在(仅仅是value=expunged)，所以针对该key的任何读写操作依然有效(读操作遇到expunged的值会返回nil)。直到misses达到阈值，dirty往read进行迁移的时候，才会判断value为expunged的key放弃迁移，使之失效。