---
title: "Kafka源码解读(3)-传输逻辑与网络IO"
date: 2021-04-01T17:45:11+08:00
description: ""
tags: ["大数据", "中间件", "Kafka"]
categories: "Kafka源码解读"
draft: true
---

Kafka的生产者，使用了消息队列的方式来实现。

1. KafkaProducer#doSend

2. RecordAccumulator#append

   1. 在acc内部为每个topic-p维护一个队列batch

      1. 在往batch添加了数据之后, 得到一个FutureRecordMetadata, 用于异步执行回调方法
      2. thunks是batch内管理FutureRecordMetadata的容器

   2. 每个batch最终由MemoryRecordsBuilder来管理数据暂存到内存对象MemoryRecords中. (最终是通过ByteBuffer.

      HeapByteBuffer

      来保存)

      1. 使用MemoryRecordsBuilder是为了更好地打包数据, 缓存在同一个batch中的数据最终可能通过分裂和合并变成(build)多个包进行发送

发送工作由Sender线程的循环来实现. Sender实现了Runnable接口, 在run方法中while(!close)反复尝试从RecordAccumulator中获取已经就绪准备发送的数据. 在最终通过`NetworkClient`实际发送之前, Sender做的主要是数据的预处理工作. 包括:

# Sender.java

1. 获取待发送的数据所属node集合(`RecordAccumulator#ready`)
2. 检查目标node连通性并确定partition leader的元数据存在(不存在则标记更新[元数据相关可参看[系列前文](https://www.gogodjzhu.com/201908/kafka-code-review-2-producer-metadata/)])
3. 获取待发送的数据(`RecordAccumulator#drain`)
4. 处理超时请求(`RecordAccumulator#expiredBatches, RecordAccumulator#deallocate`)
5. 整合发送到同一个node的多个batch，分别封装为`ProduceRequest`交给NetworkClient执行发送(`NetworkClient#send(ClientRequest request, long now)`)

# NetworkClient.java

此类封装了网络io相关的工具，是`Sender`的内部属性，对外提供了创建/断开/判断连接的方法，以及`对所有请求/响应的统一处理方法`。

1. 方法入口`NetworkClient#send(ClientRequest request, long now)`
   *注意这里接收的是ClientRequest实例，内部封装了针对不同版本的消息构造方法*
2. 将ClientRequest封装为InFlightRequest保存到inflights容器中
   *InFlightRequest是已发送尚未返回的请求，封装了请求本身及回调需要的一些信息。请求响应达到时，需要从infligts容器中移除此Request*
3. 调用`kafka.Selector#send`方法处理网络发送

# Selector.java

> 注意这里指的是`org.apache.kafka.common.network.Selector`而不是nio包中的`java.nio.channels.Selector`。此类是在common包中定义的核心类，不仅Producer，Server端同样用此类来处理网络io操作。为避免歧义，下面若不加说明，Selector均指kafka.Selector，nioSelector用于区别。

`Selector`实现了`Selectable`接口，内部通过封装`nioSelector`和`KafkaChannel`来实现网络操作。内部主要的属性和公共方法为：

```java
// nioSelector
private final nio.Selector nioSelector;
// 所有建立连接的channel都被封装为kafkaChannel并以唯一id维护在map内，
// 发送数据的时候需要制定目标channel的id
private final Map<String, KafkaChannel> channels;
// 已完成接收/发送的消息
private final List<Send> completedSends;
private final List<NetworkReceive> completedReceives;
// 实现大消息分段接收的临时容器
private final Map<KafkaChannel, Deque<NetworkReceive>> stagedReceives;

// 作为客户端创建连接到address，异步方式连接
void connect(String id, InetSocketAddress address, ...){...}
// 服务端调用，将Acceptor完成连接的channel注册到Selector做维护，
// 此方法Producer未使用
void register(String id, SocketChannel socketChannel){...}
// 发送数据，准确说是将待发数据添加到目标KafkaChannel，执行发送的时机在poll方法
void send(Send send){...}
// Selector类提供的所有io方法都是非阻塞的，通过nioSelector事件key的方式来回调
// 执行真正的发送/接收/连接动作
void poll(long timeout);
```

具体地，在NetworkClient调用Selector的`send(Send)`方法，将带发送的消息`Send`添加到目标KafkaChannel的缓存后，回到Sender类的run方法调用`poll()`真正执行发送。Selector#poll方法包含了发送/接收消息的大部分逻辑，我们着重来看看。

## 通信细节

以Producer的逻辑举例，Sender线程每次循环都会调用NetworkClient的poll()方法，此方法并不直接处理底层的网络操作，而是通过调用Selector#poll()方法来处理io。网络传输完毕的Send/Receive对象都会保存在缓存(`completedSends`, `completedReceives`)中，在poll方法执行结束后紧接着就是处理这些已完成的Send/Receive对象。

```java
// NetworkClient.java
public List<ClientResponse> poll(long timeout, long now){
  //...
  this.selector.poll(Utils.min(timeout, metadataTimeout, 
    requestTimeoutMs));
  
  long updatedNow = this.time.milliseconds();
  List<ClientResponse> responses = new ArrayList<>();
  // 处理已发送的消息(准确的说是处理已发送且不期待响应的数据，对于那些有响应的数
  // 据会在下面的handleCompletedReceives方法中处理)
  handleCompletedSends(responses, updatedNow);
  // 处理读取结果
  // 注意是所有的响应都在这里处理，包括元数据更新
  handleCompletedReceives(responses, updatedNow);
  handleDisconnections(responses, updatedNow);
  handleConnections();
  handleInitiateApiVersionRequests(updatedNow);
  handleTimedOutRequests(responses, updatedNow);
  completeResponses(responses);

  return responses;
}


// Selector.java
public void poll(long timeout) throw IOException{
  // NetworkCilent每次调用poll()方法之后都会通过handleXXX方法处理掉已完成传
  // 输的完整消息，这里把它们清除掉，防止重复处理
  clear();
  
  // ...
  // 调用nioSelector的select方法, 等待新事件的发生

  int readyKeys = select(timeout);
  if (readyKeys > 0 || !immediatelyConnectedKeys.isEmpty()) {
    // 处理新事件
    pollSelectionKeys(this.nioSelector.selectedKeys(), 
      false, endSelect);
    // 处理immediatelyConnected Channel的初始化, 注意第二个参数为true以区分
    pollSelectionKeys(immediatelyConnectedKeys, true, endSelect);
  }
}

void pollSelectionKeys(Iterable<SelectionKey> selectionKeys,
                               boolean isImmediatelyConnected,
                               long currentTimeNanos) {
  while (iterator.hasNext()) {
    // 遍历所有新事件
    SelectionKey key = iterator.next();
    iterator.remove();
    // 获取新所属的KafkaChannel
    KafkaChannel channel = channel(key);
    // ...
    if (channel.isConnected() && !channel.ready())
      channel.prepare(); // 连接建立，处理鉴权，修改状态
    /**
     * 可读事件, 处理读操作
     * stage n. 阶段；舞台；戏剧；驿站
     * 在这里解释为暂存, 由于单个Receiver可能无法一次完成读取, 故设计了
     * {@link this#stagedReceives}, 这里只负责将stageReceive不断添加到
     * map
     **/
    if (channel.ready() && key.isReadable() && 
      !hasStagedReceive(channel)) {
        NetworkReceive networkReceive;
        // 循环从channel中获取数据放入全局缓存
        // 这里有一个很重要的设计！！
        // 一个请求的响应一次性全部返回，并且不会跟其余请求的响应粘包
        while ((networkReceive = channel.read()) != null)
          addToStagedReceives(channel, networkReceive);
    }

    /**
     * 可写事件, 处理写操作
     * 每次调用write方法可能只发送send中的部分, 发送进度在send内部维护.
     * 通过{@link Send#completed()}方法判断Send是否完整完成
     */
    if (channel.ready() && key.isWritable()) {
      Send send = null;
      try {
        // 当channel通过setSend()方法设置了待发送消息后，此方法执行发送
        // 操作并返回刚发送的send对象, 否则返回null
        send = channel.write();
      } catch (Exception e) {
        sendFailed = true;
        throw e;
      }
      if (send != null) {
        // 发送了send数据，将发送过的send对象添加到completedSends
        this.completedSends.add(send);
        this.sensors.recordBytesSent(channel.id(), send.size());
      }
    }
    // key不可用, 关闭channel
    /* cancel any defunct sockets */
    if (!key.isValid())
      close(channel, true, true);
  }
}
```

大致来说, Sender处理Accumulator数据的调用栈是:

1. Sender整理/校验/合并同node数据
2. NetworkClient报文格式化/版本校验，处理完整的消息
3. Selector缓存Map(node -> KafkaChannel)，执行io操作，封装消息实体
4. KafkaChannel维持单个node的连接, 处理io的底层工作

至此，我们介绍了kafkaProducer发送数据的底层大致逻辑。 当然还有协议格式，粘包拆包问题的解决，内存管理等细节未曾深入，这个留待以后写一篇专门的文章来解释。