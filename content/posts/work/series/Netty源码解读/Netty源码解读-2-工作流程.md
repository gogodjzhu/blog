---
title: "Netty源码解读(2)–工作流程"
date: 2020-01-30T22:32:31+08:00
description: "本章结合代码介绍了Netty的核心组件，及使用Netty进行网络通信的主要流程"
featured_image: ""
images: [""]
tags: ["网络", "中间件", "JAVA", "Netty"]
categories: "Netty源码解读"
draft: false
---

# NIO的工作流程

在nio的`SelectionKey`类中，有且仅有四种事件类型：*`OP_READ`*，*`OP_WRITE`*，*`OP_CONNECT`*，*`OP_ACCEPT`*。其中`OP_ACCEPT`表示ServerSocketChannel(服务端)接收到一个新的连接请求，在SocketChannel(客户端)中无此事件；`OP_CONNECT`表示SocketChannel(客户端)发起的连接请求已被对端节点处理，`OP_READ`, `OP_WRITE`分别是读写请求，在ServerSocketChannel(服务端)无此事件。

基于此设计，在编写nio代码时需要开发者频繁地调整`Selector.interestOps(OP_XXX, Channel)`，并对Channel进行相应的读写操作，这是nio的工作流程，灵活但繁琐。



# Netty的工作流程

类似于nio，netty也把网络传输定义成的事件，但netty的事件不需要通过Selector注册后主动扫描。netty把各种事件封装为对应的入参，并回调对应事件方法。并采用责任链模式，让事件在链上传递，链上的每个节点处理各自关心的事件，使业务开发模块化。具体地，责任链上的节点被称为`Handler`，预先定义了所有可能发生的事件回调方法，子类只需要重写关心的方法即可。

- Handler接口预先定义了IO事件的回调方法

  ![img](http://minio.gogodjzhu.com/images/20210403_151254_09d6762d-7345-4dd9-b874-b0e56273841b.png)

具体地，在设计上Netty包括以下几个核心的组件：

- `EventLoop` 处理绑定Channel的所有I/O操作的实体。继承了Executor，可以简单理解成一个独立的工作线程。
- `EventLoopGroup` 包含多个EventLoop的线程池。
- `ChannelPipeline` 一个连接建立后，会创建一个对应的Pipeline实例，Piepeline内按顺序包含若干handler，当有IO事件触发之后会依次回调Handler上面的方法。可以将Pipeline理解成对责任链的抽象。
- `ChannelHandler`是扩展功能的核心，通过组合不同的Handler实现各种业务应用。简单地Handler分为两种类型:`Inbound`和`OutBound`(也可以兼而有之)分别处理入站消息(对端发给当前Enpoint)和出站消息(当前Endpoint发给对端)。
- `ChannelHandlerContext` 为Handler提供上下文环境。
- `ByteBuf` 是netty提供的ByteBuffer升级版缓存，支持读写指针、堆外内存、对象池等特性。这是netty高性能很重要的一个原因。

下图展示了Netty核心组件的协作流程：

![img](http://minio.gogodjzhu.com/images/20210403_152603_da8a462f-a4ed-42d3-8af0-4676d35bdaa1.png)

# 一个Netty示例应用

下面借用Netty源码中的一个例子，实现简单的Echo服务。服务端负责将所有客户端发来的信息原样返回给客户端。让我们开始吧:)

## 服务端

```java
# Server

public static void main(String[] args) throws Exception {
    // Configure the server.
    EventLoopGroup bossGroup = new NioEventLoopGroup(1);
    EventLoopGroup workerGroup = new NioEventLoopGroup();
    try {
        ServerBootstrap b = new ServerBootstrap();
        b.group(bossGroup, workerGroup)
                .channel(NioServerSocketChannel.class) // 创建NioServerSocketChannel类型的Channel实例
                .childHandler(new ChannelInitializer<SocketChannel>() {
                    @Override
                    public void initChannel(SocketChannel ch) throws Exception {
                        ChannelPipeline p = ch.pipeline();
                        p.addLast(new ChannelInboundHandlerAdapter(){ // 添加一个自定义InboundHandler
                            // 当有可读数据进来的时候原样发送回去
                            @Override
                            public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
                                ctx.writeAndFlush(msg);
                            }
                        });
                    }
                });

        // Start the server.
        ChannelFuture f = b.bind(8007).sync();

        // Wait until the server socket is closed.
        f.channel().closeFuture().sync();
    } finally {
        // Shut down all event loops to terminate all threads.
        bossGroup.shutdownGracefully();
        workerGroup.shutdownGracefully();
    }
}
```

在Server中端，通过`.channel(NioServerSocketChannel.class)`指定这是基于NIO创建的网络应用。通过一个特殊的`ChannelInitializer`实例初始化ChannelPipeline，在这个Pipeline中我们添加一个原样返回数据的`ChannelInboundHandlerAdapter`。

由于Netty的所有操作都是异步非阻塞的，所以通过`Future#sync()`方法强制等待操作顺序完成。当然，这里的Future不是`java.util.concurrent`包的Future，netty基于后者实现了自己的Future类。主要增加了以下几个功能需要你注意：

- 成功/失败的标记 (通过`isSuccess()`)
- 支持动态增加执行结束后的回调(通过`addListener()`)
- 支持通过`sync()`/`await()`方法阻塞等待异步操作完成

在Future的基础上，netty还定义了一个`Promise`接口以支持对异步操作的结果进行写入更新（在异步操作已经完成后，拥有Promise实例的线程仍可对其进行修改）（通过trySuccess(V),tryFailure(Throwable)方法）。下图是netty提供的异步回调接口继承图：

![img](http://minio.gogodjzhu.com/images/20210403_152518_96bf38a7-d01a-44f4-a173-a05aca8b4277.png)



## 客户端

```java
# Client

public static void main(String[] args) throws Exception {
    // Configure the client.
    EventLoopGroup group = new NioEventLoopGroup();
    try {
        Bootstrap b = new Bootstrap();
        b.group(group)
         .channel(NioSocketChannel.class) // 创建NioSocketChannel类型的Channel实例
         .handler(new ChannelInitializer<SocketChannel>() {
             @Override
             public void initChannel(SocketChannel ch) throws Exception {
                 // pipeline
                 ChannelPipeline p = ch.pipeline();
                 p.addLast(new ChannelInboundHandlerAdapter(){ // 添加一个自定义Handler
                     // channel active后回调此方法, 发送一个简单的字符串'ping'给Server
                     @Override
                     public void channelActive(ChannelHandlerContext ctx) throws Exception {
                         ByteBuf byteBuf = Unpooled.buffer();
                         byteBuf.writeBytes("ping".getBytes(CharsetUtil.UTF_8));
                         ctx.writeAndFlush(byteBuf);
                     }

                     // 有可读数据时回调此方法, msg是接收的数据实体
                     @Override
                     public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
                         ByteBuf byteBuf = (ByteBuf) msg;
                         System.out.print(byteBuf.toString(CharsetUtil.UTF_8));
                     }

                     // 读取操作完成后回调此方法, 关闭channel
                     @Override
                     public void channelReadComplete(ChannelHandlerContext ctx) throws Exception {
                         ctx.close();
                     }
                 });
             }
         });

        // Start the client.
        ChannelFuture f = b.connect("127.0.0.1", 8007).sync();

        // Wait until the connection is closed.
        f.channel().closeFuture().sync();
    } finally {
        // Shut down the event loop to terminate all threads.
        group.shutdownGracefully();
    }
}
```

在Client端，创建Channel的实例类型是`NioSocketChannel.class`，表明底层是一个java nio 的SocketChannel在处理网络操作。在自定义的Handelr中，Client端比Server更多地重写了几个方法：

1. `:channelActive()`channel连接在进入active状态后回调此方法，执行一个主动的消息发送。channel在可以执行读写操作前会依次进入两个状态: register/active。在[构建连接]一文会详细介绍连接相关内容，这里可以简单理解成客户端连接上服务端后执行此回调方法
2. `:channelRead()` chennel有可读数据时回调此方法，需要注意的是由于粘包/拆包的原因，依次调用此方法可能无法完整地获取一次请求的所有内容。后续文章也有专门的解析。
3. `:channelCompelete()` 完成一次完整的数据读取(对应对端的一次完整write()操作)之后，会回调此方法。这里我们简单将连接断开。

通过EchoServer这个简单的例子可以发现Netty几乎完全帮我们屏蔽了底层的网络传输细节。我们这里使用的是NIOChannel进行通信，但是在应用中对nio核心的组件包括channel, selector, SelectionKey, ByteBuffer都完全被封装起来，开发者的注意力只需要放在如何编写Handler实现逻辑即可。

# 小结

本文对比nio简单介绍了netty的工作流程，并以一个EchoServer的例子演示如何基于netty开发应用程序。可以看到开发netty应用程序是非常简单且直观的，这首先得益于它把网络事件以责任链模式封装起来，开发人员只需要编写业务关心的对应handler即可。此外还介绍了Netty的几个核心组件，以及它们是如何配合工作的。内容比较浅显，但却是稍后展开讨论的基础，希望你能有所收获！