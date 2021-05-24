---
title: "Kafka源码解读(2)-集群元数据"
date: 2021-04-01T17:45:11+08:00
description: ""
tags: ["大数据", "中间件", "Kafka"]
categories: "Kafka源码解读"
draft: true
---

> KafkaProducer是客户端, 它的服务端是Kafka的整个集群. 要想获取集群提供的服务, 第一步自然是掌握集群的状态信息
>
> 即：我是谁? 我在哪? 我在干什么?

# Metadata

集群元数据的获取, 这是集群客户端必须解决的第一个问题, 从第一次出现KafkaProducer这个类的[提交记录](https://github.com/gogodjzhu/kafka/blob/269d16d3c915d09f650ae32aa81542bd8522ca68/clients/src/main/java/kafka/clients/producer/KafkaProducer.java)开始, 已经在类中定义了`Metadata`变量:

```java
public class KafkaProducer implements Producer {
  private final Metadata metadata;
}
```

而在[最早版本的Metadata](https://github.com/gogodjzhu/kafka/blob/269d16d3c915d09f650ae32aa81542bd8522ca68/clients/src/main/java/kafka/clients/producer/internals/Metadata.java)中, 已经定义了几种最基础的数据:

```java
public final class Metadata {
  // 两次更新之间的最小时间间隔, 防止太频繁的请求
  private final long refreshBackoffMs;
  // 多久全局更新一次meta信息(这是时间触发条件, 还有事件触发条件)
  private final long metadataExpireMs;
  // 上一次更新时间(也包含更新失败的情况, 新版本添加了上次成功更新时间的参数,
  // 用来实现更精细的控制)
  private long lastRefresh;
  // 集群信息, 注意
  // 1. 这里的集群信息不是完整的, 每次更新可能会按需新增信息
  // 2. 由于缓存信息的缘故, 读取的可能有延迟.
  private Cluster cluster;
  // 需要更新标记. 更新的逻辑是先标记, 再更新
  private boolean forceUpdate;
  // this.cluster中维护的topic集合
  private final Set<String> topics;
}
```

[*Cluster代码*](http://xn--cluster-bx3k98as0ag224azf7boghv5g./)*这里不贴了. 主要是作为POJO类存在, 保存了broker节点信息(List), 每个topic下所有的Partition信息(Map<String, List<>>), 每个broker节点下所有的Partition信息(Map<Integer, List<>>)等等.*

按照Metadata 的Javadoc所说, 这是*`A class encapsulating some of the logic around metadata.`* 封装了关于元数据的一些逻辑(实际上还包括数据, 在Cluster中), 本身并不直接获取元数据, 更新操作是通过暴露`public update(Cluster c)`方法来实现的. 

> ###### 线程安全
>
> Metadata中的所有方法均用`synchronize`方法修饰以保证线程安全, 而为一个对象属性`Cluster`的更新使用`this.cluster = cluster;`的方法修改, 也可以保证不会出现并发问题

# 更新数据

既然Metadata不处理数据的获取, 那么谁来做? 

![img](http://minio.gogodjzhu.com/images/20210418_180757_060cb6dc-ee93-42e8-a7d5-570de223ac8a.jpeg)

使用idea的调用查看, 筛选除去了非当前modual的调用

看到有4处调用, 其中`KafkaAdminClient`是管理员api, `KafkaConsumer`是消费者端api. 这两个暂不讨论. 剩下的两个:

1. `KafkaProducer`, 定位过去发现是启动Producer做的初始化. 它通过`Cluster#bootstrap(List<InetSocketAddress>):Cluster`方法获取初始化的Cluster对象, 但其中并不包含任何topic相关的信息, 只获取启动时配置的broker列表
2. 2.`NetworkClient`是关键, 这个类负责客户端(包括Producer和Consumer)处理网络请求/响应, 具体我们在后续网络通信部分会继续讨论. 这里我们只要知道metadata是通过response回调处理更新的即可. 那么下一个问题是, 是谁发送的这个更新请求呢?

跟踪`needUpdate`属性, 因为它标记当前metadata是否需要更新. 同样分两步:

1. 谁修改的needUpdate参数为true.
2. 谁判断needUpdate并发送更新请求

追踪代码发现, 在Producer内标记needUpdate=true主要落在`KafkaProducer#waitOnMetadata(topic:String, partition:Integer, maxWaitMs:long) : ClusterAndWaitTime` 这个方法里, 它在每次执行发送之前都会被执行一次.

这个方法首先会尝试从metadata缓存中获取指定topic下某个partition的信息, 如果缓存中存在目标信息, 直接将`metadata#Cluster`对象封装返回(那么这次调用并没有实际更新metadata). 如果获取失败, 则将needUpdate修改为true, 之后就`wait`等待更新的发生. 如果超过了`maxWaitMs`未获取到目标数据, 抛出异常.

而发送更新metadata请求的是`Sender(extends Runnable)`常驻线程. 作为一个独立的线程运行, Sender反复调用[poll方法](https://github.com/gogodjzhu/kafka/blob/6e4e2d2294ddfd3145dc706e220357d97133b935/clients/src/main/java/org/apache/kafka/clients/NetworkClient.java#L429)查看是否有可写入事件发生, 并在发现需要更新时将metadata更新请求发送出去. 

最后一个问题, 这么多的broker, 发给谁? 答案是负载最低的kroker. Kafka维护了一个按brokerId 分组的Map<Integer, Deque> `InFlightRequests`, 保存的是每个broker等待返回的请求列表. 选择等待返回最少的作为负载最低的broker.

> 负载最低
>
> 这是一个很有趣的问题, 寻找负载最小的节点, 通常的做法是获取到所有节点的元数据, 在元数据中包含了每个节点的负载信息(例如内存使用率). 但是在KafkaProducer中, 一方面因为这里的metadata本身不包含负载信息, 客观上无法做负载判断; 另外我们要请求的是更新的元数据, 那么目前缓存的metadata必然不是最新; 第三, producer维护的metadata是集群broker的子集, 所以也没办法获取全部broker中压力的最小者. 这里是[实现代码](https://github.com/gogodjzhu/kafka/blob/bb1ee0f6d880143359a89baeccb477fe8b9ad2a0/clients/src/main/java/org/apache/kafka/clients/NetworkClient.java#L536).

至此, 我们解释了KafkaProducer维护元数据的逻辑.