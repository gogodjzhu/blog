---
title: "Netty源码解读(1)-概念&原理"
date: 2020-01-30T22:22:31+08:00
description: "Netty源码解读系列第一章，介绍Netty诞生的背景和相关前置技术的特点和不足，阐述Netty设计的初衷和愿景"
tags: ["网络", "中间件", "JAVA", "Netty"]
categories: "Netty源码解读"
draft: false
---

Netty是Line公司的Trustin Lee(韩国)在2004年研发的Java网络应用程序框架。在进入Netty内容之前先得介绍Java网络通信的背景知识。这部分内容一带而过，阅读系列文章的你，应该通过别的渠道对基于Java基础库的网络编程有基础的了解。

## BIO & NIO & AIO

BIO / NIO / AIO 分别代表 Block I/O, Non-Blocking I/O和Asynchronize I/O。前两者的涵义是见名知意的，AIO则是号称完全异步非阻塞。但实际上AIO因为引入的复杂度远比带来的性能提升要低，所以在工程应用中很少采用。

但不论是哪一种IO，在编程思想和代码实现上都存在巨大的差异。Netty作为框架工具，提供了统一以上所有IO的统一编程接口。甚至对操作系统依赖的一些网络通信技术提供了支持，如Linux的Epoll。本系列主要围绕NIO（这也是业界使用最多的）展开，为了文章连续性，先简单介绍NIO的核心概念。

## NIO核心概念

NIO的提供的核心组件是`Channel`、`Selector`和`ByteBuffer`。其中Channel共有两种实现`ServerSocketChannel`和`SocketChannel`，底层封装了`Socket`(准确地说也分为`ServerSocket`和`Socket`两种实现)。`Socket`是JDK1.0提供的，直接用于网络通信的终端组件，通过`Socket`你可以创建`InputStream`/`OutputStream`用于网络传输，这样就是普通BIO的实现方式。但通过Stream进行的数据读写都是阻塞的(阻塞的弊端这里不做展开)。

所以从JDK1.4推出NIO的Channel，底层虽然封装了Socket，但用户不需要直接通过它来进行读写，Channel把通过Socket做的 连接/读/写 请求封装为对应的`OP_ACCEPT`/`OP_READ`/`OP_WRITE`事件。用户可以通过同一个Selector注册多个Channel的特定事件，而每个Channel底层是一个Socket即一个网络连接，故使用一个独立线程去轮询单个Selector就可以实现对多路连接的监听。这是所谓的事件驱动。

NIO还有一个重要的特点是异步。在基于Channel+Selector实现了IO事件的监听后，当有读写事件触发，工作线程`Worker-Thread`跟Channel的数据交换会通过ByteBuffer进行中转，避免了高速的CPU等待缓慢的IO(即阻塞)。

- Nio应用结构图

  ![img](http://minio.gogodjzhu.com/images/20210403_153127_f56a4957-11d9-42cd-ae0a-c5f9a8318de2.png)

## Netty登场

前面的介绍可以看到 ，NIO的设计比BIO优化不少，事实上NIO也已经成为Java生态中的网络通信标准工具。那么为什么要有Netty？答案是，NIO在网络通信上做得很好，但是面对应用开发却很不友好。且不说api本身的复杂度过高，以读操作为例，在完成数据接收之后用户得到的也只是保存在ByteBuffer内的字节，需要经过拆包、校验、解析等工作方才得到Java对象用于业务运算。这些无疑是在诸多NIO应用中的重复工作，而且容易出错。而这，恰恰是Netty提供的，简化网络开发工作，提供开发者友好的统一编程框架。

- 统一各种网络传输类型(BIO,NIO,~~AIO~~)
- 事件驱动开发模型(这里的事件跟NIOSelector的事件类似，但netty做的更细致且优雅)
- 可配置的线程模型，可以轻松实现三种线程模型：
  - `new NioEventLoopGroup(1)` 单线程处理所有操作
  - `new NioEventLoopGroup()` 多线程(默认跟cpu核心数等比)处理所有请求
  - bossGroup / workerGroup 分工。两个线程池分别处理：
    1. 握手/认证等连接建立工作
    2. 连接建立后的数据交换工作
- 更高的性能(缓存/零拷贝)，更低的性能消耗(对象池/内存复用)
- 多连接之间相对公平的流量控制
- 丰富的开箱即用工具 和 完善的文档及例子