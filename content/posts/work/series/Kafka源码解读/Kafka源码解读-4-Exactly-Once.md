---
title: "Kafka源码解读(4)-Exactly Once"
date: 2021-04-01T17:45:11+08:00
description: ""
tags: ["大数据", "中间件", "Kafka"]
categories: "Kafka源码解读"
draft: true
---

> 大家共同的指标，是我们要做到 Exactly-Once 才发布1.0版本，这才能使 Kafka 作为一个成熟的流数据平台持续发展。
>
> —— [Kafka 的七年之痒](https://juejin.im/entry/5be94dbaf265da61616e393f)



# 幂等Producer

`enable.idempotence=true` 以开启。
实现：
在Producer端：
\1. retry次数设为无限
\2. 创建一个TransactionManager，用于控制事务。默认未配置`ProducerIdAndEpoch`(PID)。
\3. 在Sender线程启动后，发现TransactionManager存在且无ProducerIdAndEpoch，尝试发送发送`InitProducerIdRequest`请求给server，并阻塞等待返回成功，将ProducerIdAndEpoch的值设置到`TransactionManager`中。
\4. Sender继续正常发送消息的逻辑。每次发送一个新的batch时，会生成一个Topic-Partition单调递增的`sequenceNumber` 辅以 前面的`ProducerIdAndEpoch` 用于唯一标定`一个Producer向一个TopicPartition发送的一条消息`。重试发送同一条消息`sequenceNumber`不会改变，这样就为Broker去重消息提供了依据。
在Broker端：
在收到消息之后，会取出`sequenceNumber`进行校验。值得一提的是，由于在幂等配置开启后，inflight消息的数量被控制为1，因此Broker端只需要记下每个Producer发送的最后一个消息的sequenceNumber，新消息只需与上一条消息的sequenceNumber比较是否与之相同即可判断是否重复（重复的唯一来源就是重试）。

# Exactly-Once Producer

在做分析具体的kafka实现之前，我们先跳出细节，讨论下要实现Exactly-Once语义需要考虑什么问题，用什么思路去解决。

要知道的是，我们已经通过`幂等性Producer`实现了`Producer(1)- Batch(1)-TopicPartition(1)`之间传输的Exactly-Once语义，接下要讨论的是如何实现单Producer在同一事务下跨Batch跨TopicPartition的Exactly-Once语义。

## 场景介绍

一个常见的事务场景为：从某些topic消费数据，经过业务处理之后再写入另外的若干topic供下游消费。事务的需要保证读操作 和 写操作 要么同时成功，要么同时失败。我下面提供的代码，是截取自[Kafka官方](https://cwiki.apache.org/confluence/display/KAFKA/KIP-98+-+Exactly+Once+Delivery+and+Transactional+Messaging#KIP-98-ExactlyOnceDeliveryandTransactionalMessaging-AddOffsetsToTxnRequest)的一段demo，实现的就是这种常见的。

```java
public class KafkaTransactionsExample {
  
  public static void main(String args[]) {
    // 创建consumer
    KafkaConsumer<String, String> consumer = new KafkaConsumer<>(consumerConfig);
    // 创建Producer，配置中必须包含'transactional.id'
    KafkaProducer<String, String> producer = new KafkaProducer<>(producerConfig);
 
    // 在producer做任何操作之前，需要先调用init方法. 相同transactional.id的
    // producer如果存在执行状态, 这里会进行恢复或者丢弃
    producer.initTransactions();
     
    while(true) {
      // consumer拉取的每个batch作为一个事务进行读写
      ConsumerRecords<String, String> records = consumer.poll(CONSUMER_POLL_TIMEOUT);
      if (!records.isEmpty()) {
        // 开启一个事务
        producer.beginTransaction();
         
        // 遍历一个batch内的所有record通过producer依次写入目标topic
        List<ProducerRecord<String, String>> outputRecords = processRecords(records);
        for (ProducerRecord<String, String> outputRecord : outputRecords) {
          producer.send(outputRecord);
        }
         
        // 提交consumer消费的最新offset, 注意这里是通过producer来提交, 这样
        // 保证了在一个producer 事务内执行读写操作
        sendOffsetsResult = producer.sendOffsetsToTransaction(getUncommittedOffsets());
        
        // 提交事务 
        producer.endTransaction();
      }
    }
  }
}
```



## 实现细节



### Producer启动流程

添加了`transactional.id`配置之后，Producer在实例化的过程中会创建一个`TransactionManager`对象（文本部分使用TxnManager代替）。这个TxnManager是实现ExactlyOnce的关键，观察它的构造方法：

```java
# TransactionManager.java

public TransactionManager(String transactionalId, 
    int transactionTimeoutMs, long retryBackoffMs) {
  // 用于标识一个Producer的关键ProducerId并未指定
  this.producerIdAndEpoch = new ProducerIdAndEpoch(NO_PRODUCER_ID, NO_PRODUCER_EPOCH);
  // 保存所有sequenceNumber, Producer发送的每个Batch都有一个单调递增的Id用于标识(结合TopicPartition/ProducerId一起)
  this.sequenceNumbers = new HashMap<>();
  this.transactionalId = transactionalId;
  this.logPrefix = transactionalId == null ? "" : "[TransactionalId " + transactionalId + "] ";
  this.transactionTimeoutMs = transactionTimeoutMs;
  // transactionCoordinator 所在的Node
  this.transactionCoordinator = null;
  // consumerGroupCoordinator 所在的Node
  this.consumerGroupCoordinator = null;

  /* 为事务设计的几种TopicPartition概念，对不同类型的TopicPartition操作需要不同的机制来保证事务特性 */
  this.newPartitionsInTransaction = new HashSet<>();
  this.pendingPartitionsInTransaction = new HashSet<>();
  this.partitionsInTransaction = new HashSet<>();
  this.pendingTxnOffsetCommits = new HashMap<>();

  // 事务操作队列
  this.pendingRequests = new PriorityQueue<>(10, new Comparator<TxnRequestHandler>() {
      @Override
      public int compare(TxnRequestHandler o1, TxnRequestHandler o2) {
          return Integer.compare(o1.priority().priority, o2.priority().priority);
      }
  });

  this.retryBackoffMs = retryBackoffMs;
}
```

你会发现，部分Producer自己可以确定的参数在实例化的时候就已经创建，比如：

- `sequenceNumbers`是Prducer内部维护的batch唯一标识
- `transactionalId`是用户在`transactional.id`配置的
- `transactionTimeoutMs`是事务超时时间，在配置文件定义
- `pendingRequests`是Producer**待发送**的所有事务请求的队列

而其中有两个参数却未初始化：

- `producerIdAndEpoch`是在集群中确定唯一Producer的标识。由于是集群内唯一，那么就不能由Producer自己来确定，需要由Broker来统一分配。
- `transactionCoordinator` 和 `consumerGroupCoordinator` 两个coordinator都是用来处理特殊任务的`Broker`节点，其中Producer就是向transactionCoordinator去申请producerIdAndEpoch的。

因此两个参数的原因，kafkaProducer在创建之后，必须调用`initTransaction()`方法，通过网络请求来申请这两个参数资源：

```java
# KafkaProducer.java

public void initTransactions() {
  // 检查TransactionManager实例存在(配置了transactional.id即会创建)
  throwIfNoTransactionManager();
  // 构造请求添加到transactionManager.pendingRequests队列
  TransactionalRequestResult result = transactionManager.initializeTransactions();
  // 主动唤醒sender, 处理transactionManager.pendingRequests的消息
  sender.wakeup();
  // 等待请求执行结束
  result.await();
}

public synchronized TransactionalRequestResult initializeTransactions() {
  ensureTransactional(); // 检查transactional.id配置存在
  transitionTo(State.INITIALIZING); // 事务状态检查与切换
  setProducerIdAndEpoch(ProducerIdAndEpoch.NONE);
  this.sequenceNumbers.clear(); // 重置sequenceNumber
  /** 构造&发送InitProducerIdRequest用于请求PID */
  InitProducerIdRequest.Builder builder = new InitProducerIdRequest.Builder(transactionalId, transactionTimeoutMs);
  InitProducerIdHandler handler = new InitProducerIdHandler(builder);
  enqueueRequest(handler); /** 底层实际未发送，仅仅添加到消息队列，由{@link Sender#run(long)} */
  return handler.result;
}
```

你会发现`KafkaProducer#initTransactions()`是阻塞等待请求完成的，所以在此方法返回的时候，`producerIdAndEpoch`就已经申请下来了。这是如何做到的？要回答这个问题，就需要把目光移到Sender线程，看`TransactionManager#enqueueRequest(handler)`方法把请求添加到消息队列之后，是怎样发送出去，又是如何处理响应的。

在[KafkaProducer-传输逻辑&网络IO](https://www.gogodjzhu.com/201910/kafka-code-review-3-transfer-and-io/)当中，我已经给你介绍过了Sender发送普通消息的流程，你已经知道Sender线程有一个循环执行的`run()`方法，此方法对事务Producer(即TxnManager不为null)会有特殊的处理流程：

```java
# Sender.java

void run(long now) {
  // 带事务管理的发送
  if (transactionManager != null) {
    /** 带事务管理的Producer, 需要先做确保事务请求{@link TransactionManager#pendingRequests}优先处理 **/
    if (!transactionManager.isTransactional()) {
      // transactionManager实例存在 但 没有配置transactional.id 标识这是一个幂等producer, 需要保证它存在producerId
      maybeWaitForProducerId();
    } else if (transactionManager.hasInFlightRequest() || maybeSendTransactionalRequest(now)) {
      // hasInFlightRequest()==true标识目前事务还有请求未响应
      // maybeSendTransactionalRequest()==true表示有事务请求需要优先发送
      client.poll(retryBackoffMs, now);
      return;
    }
    // ... 省略错误处理
  }
  
 /** 常规的Producer Data消息 */
  long pollTimeout = sendProducerData(now);
  client.poll(pollTimeout, now);
}
```

再往下跟踪代码，你会发现这里的特殊处理就是Sender会优先发送TxnManager内部请求队列的消息，这个队列就是：`pendingRequests : PriorityQueue<TxnRequestHandler>`，它保存了事务相关的请求，他们包括：

- `FindCoordinatorRequest`
- `InitProducerIdRequest`
- `AddPartitionsToTxnRequest`
- `AddOffsetsToTxnRequest`
- `TxnOffsetCommitRequest`
- `EndTxnRequest`

回到现在聊的`FindCoordinatorRequest`，`Sender#maybeSendTransactionalRequest()`方法会检查TxnManager的请求队列，发现需要`TransactionCoordinator`的请求时会中断此请求(重试机制会重新发送)，并将一个`FindCoordinatorRequest`插入请求队列用于获取此请求需要的TransactionCoordinator。

对于一个事务，TxnManager的第一个请求是`InitProducerIdRequest`，它就是一个需要需要`TransactionCoordinator`的请求。那么到现在，事务Producer的启动流程就完整了：

1. 添加`transactional.id`配置启动
2. 调用`KafkaProducer#initTransactions()`方法, `InitProducerIdRequest`进入TxnManager请求队列(响应异步处理)。
3. Sender线程扫描TxnManager的请求队列准备发送请求。
4. Sender线程扫描到`InitProducerIdRequest`请求，由于需要TransactionCoordinator，插入`FindCoordinatorRequest`请求到请求队列。(InitProducerIdRequest则失败等待下次重试)
5. `FindCoordinatorRequest`响应成功, Server将分配给当前Producer的TransactionCoordinator信息返回。
6. `InitProducerIdRequest`重试发送，此时已有`TransactionCoordinator`信息，故将`InitProducerIdRequest`请求发送给这个TxnCoordinator。
7. `InitProducerIdRequest`响应成功，`ProducerIdAndEpoch`信息获取完毕，Producer初始化成功。

> TransactionCoordinator的获取策略
>
> 由于TransactionCoordinator所做的工作本质上是对事务状态topic(__transaction_state)的操作，而操作Topic最终落到每个TopicPartition Leader上。因此，kafka将某个事务的TxnCoordinator线程运行在事务状态topic(__transaction_state)保存此事务状态的Partition的Leader broker上。那么具体由哪个Partition来保存某个事务的状态？Kafka使用`hash(transactional.Id)%事务状态topic的分区数(默认40)` 来计算得到。

![img](http://minio.gogodjzhu.com/images/20210418_181143_8b1a08ee-fb6e-4d18-951a-f084907deaac.png)事务 – 事务状态topic – TxnCoordinator之间的关系.
其中绿色的独立进程, 橙色表示一个线程, 而蓝色则只是一个抽象概念

### Producer发送数据到TopicPartition

Producer启动之后，就可以开始发送数据了。每个完整的事务均以`KafkaProducer#beginTransaction()`开始，以`KafkaProducer#commitTransaction()`结束。

在Producer端调用了beginTransaction()方法之后，会修改本地的状态标识事务开始。(但在TxnCoordinator中，此事务的状态并未改变，直到Producer发送第一个事务请求，才开始变为`Ongoing`)

> 事务的状态(TransactionCoordinator中记录的事务状态)
>
> – Empty
> – Ongoing
> – PrepareCommit
> – PrepareAbort
> – CompleteCommit
> – CompleteAbort
> – Dead
> – PrepareEpochFence

事务开始之后可能发送多个`AddPartitionsToTxnRequest`请求，用于将当前事务操作涉及的分区告知TxnCoordinator

最后会通过一个`EndTxnRequest`请求通知TxnCoordinator此事务成功/失败。之后TxnCoordinator再异步地将事务的提交/回滚操作组织成Marker通过发送给事务涉及的TopicPartition完成事务。