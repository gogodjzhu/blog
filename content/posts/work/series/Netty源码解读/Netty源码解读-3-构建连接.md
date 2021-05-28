---
title: "Netty源码解读(3)–构建连接"
date: 2020-01-31T08:12:31+08:00
description: "网络通信的基础是建立连接，本章介绍了Netty在NIO之上是如何构建连接的，开始真正地对源码进行分析，使用的demo代码还是前文提到的EchoServer。"
tags: ["网络", "中间件", "JAVA", "Netty"]
categories: "Netty源码解读"
draft: false
---

# Reactor线程模型

Reactor线程模型是常见的，用于解决`事件-响应`场景问题的设计。按照Doug Lea在《Scalable IO in Java》中的描述，Reactor线程模型的核心思想是分治，把大的问题（端对端的网络通信）分为一系列的小问题来解决。Netty也采用了Reactor线程模型来处理IO事件。

在netty中有两种线程池:`WorkerGroup`和`BossGroup`(或称之为`parentGroup`和`childGroup`)。你在EchoServer的Server端可以找到这行代码：

```java
ServerBootstrap b = new ServerBootstrap();
b.group(bossGroup, workerGroup)
```

这里即指定了两个线程池，前者bossGroup用于监听并处理连接请求，后者workerGroup用于处理连接建立后的IO请求。根据Reactor模型的设计，细分为三种模式，分别是：

- `new NioEventLoopGroup(1)` 单线程处理所有操作
- `new NioEventLoopGroup()` 多线程(默认跟cpu核心数相等)处理所有请求
- bossGroup / workerGroup 分工。

相应地，启动工具类ServerBootstrap也提供了两个和一个参数的`group()`重载方法以支持上述模式。



# 定义Channel类型

Netty支持多种网络模式，本系列文章围绕NIO展开，这在代码实现上只需要在一个地方指定即可：

```java
// server端
ServerBootstrap.channel(NioServerSocketChannel.class)

// client端
Bootstrap.channel(NioSocketChannel.class)
```

这行代码指定了创建netty.Channel实例的类型，在底层会通过一个工厂类`ReflectiveChannelFactory`保存这个Channel类的引用，在需要创建Channel实例的时候通过反射创建。

需要注意的是，这里的Channel并不是nio的Channel(但是命名非常类似，需要注意)，而是netty自己定义的一个接口。它定义类似nioChannel那样的I/O操作（包括read,write,bind,connect）但作为一个接口它并不限制底层是否一个网络Socket还是其他。典型地，netty实现了一个`EmbeddedChannel`，它实现了Channel接口，底层却无任何网络IO。如果需要改变底层实现为EmbeddedChannel（如用于测试）只需要简单修改ServerBootstrap#channel()方法的入参为`EmbeddedChannel.class`即可，其余的业务逻辑完全不受影响。

> 为了区别nio提供的Channel和netty提供的channel，后文记为nioChannel和Channel



# 建立连接

一个netty应用的网络传输本质，依赖于Channel实例的类型，在系列文章中主要围绕NIO展开，所以它的网络传输的第一步——建立连接——的核心也就是nioChannel实例的连接，更本质是TCP三次握手的过程。具体到代码层面则是`ServerSocketChannel#bind(port:int)`和`SocketChannel#connect(host:string, port:int)`方法的调用。



## ServerBootstarp#bind(port:int)

ServerBootstrap工具类提供了bind方法，用于将nioServerSocketChannel绑定到指定端口进行监听。跟随下面这个调用链，来到doBind方法：

```java
EchoServer.main(String[]) (io.netty.example.echo)
AbstractBootstrap.bind(SocketAddress) (io.netty.bootstrap)
AbstractBootstrap.doBind(SocketAddress) (io.netty.bootstrap)
```

`AbstractBootstrap.doBind()`先调用`AbstractBootstrap#initAndRegister()`方法创建Channel实例，并对此Channel做初始化工作:

```java
// AbstractBootstrap.java
final ChannelFuture initAndRegister() {
    Channel channel = null;
    try {
        // 依赖channel工厂创建channel实例, 在EchoServer例子中
        // channelFactory是基于反射实现的工厂类
        channel = channelFactory.newChannel();
        // 初始化已创建的channel
        init(channel);
    } catch (Throwable t) {
        // ...
    }

    // 将创建的channel注册到EventLoopGroup中，本质上是将channel的关注事件
    // 注册到Selector上
    ChannelFuture regFuture = config().group().register(channel);
    // ...
    return regFuture;
}
```

channelFactory在EchoServer中使用`ReflectiveChannelFactory`工厂类创建NioServerSocketChannel实例，查看工厂类的源码，可以发现它调用的是指定类即NioServerSocketChannel的无参构造方法，跟踪构造函数发现主要做以下三件事：

1. 依赖`SelectorProvider.*provider*().openServerSocketChanne()`创建nioServerSocketChannel

2. 将nioServerSocketChannel添加到属性中(准确地说是父类AbstractNioChannel的`ch`属性)以供后续操作。同时在构造实例的过程中还初始化父类的SelectionKey的初始值为`SelectionKey.OP_ACCEPT`，表示此Channel此时关心的是连接事件。

3. 在父类AbstractChannel的构造方法中调用`newUnsafe()`和`newChannelPipeline()`方法创建`Unsafe`和`ChannelPipeline`实例属性。

   - `ChannelPipeline`在前文已经交代过，是netty实现的对`责任链`的抽象。AbstarctChannel实现的方法采用`DefaultChannelPipeline`类。
     DefaultChannelPipeline包含以下几个核心属性：
     – head:AbstractChannelHandlerContext
      – tail:AbstractChannelHandlerContext
     – channel:Channel
      这几个属性构成了责任链的链式结构的基础。入站请求从head开始遍历Context, 出站请求则从tail开始遍历Context。
   - `Unsafe`是Channel接口提供的内部接口，正如其名所指，这是一个涉及到底层（包括分配堆外内存，获取本地/对端网络地址，网络IO等）的非安全操作的接口，不会暴露给用户使用。同时由于它跟底层操作有关，所以在AbstractChannel中只定义了一个`newUnsafe()`抽象方法，留待子类去实现(本例中由AbstractNioChannel实现)。

以上是创建NioServerSocketChannel的过程。回到`AbstractBootstrap#initAndRegister()`方法，在完成Channel的实例化之后，紧接着会调用`init(channel)`初始化channel。由于Channel不同，可预料初始化工作也是不同的，在AbstractBootstrap中仅定义了抽象的`init(Channel)`方法，具体实现落到了`ServerBootstrap`中。

```java
// ServerBootstrap.java

void init(Channel channel) throws Exception {
    final Map<ChannelOption<?>, Object> options = options0();
    // 使用配置项配置channel...

    // 获取channel绑定的pipeline, 本例中为 DefaultChannelPipeline 实例
    ChannelPipeline p = channel.pipeline();

    // 亦即workerGroup
    final EventLoopGroup currentChildGroup = childGroup;
    // Bootstrap的初始Handler,本例中是ChannelInitializer
    final ChannelHandler currentChildHandler = childHandler;
    
    // worker线程的配置项和参数
    final Entry<ChannelOption<?>, Object>[] currentChildOptions;
    final Entry<AttributeKey<?>, Object>[] currentChildAttrs;
    synchronized (childOptions) {
        currentChildOptions = childOptions.entrySet().toArray(newOptionArray(childOptions.size()));
    }
    synchronized (childAttrs) {
        currentChildAttrs = childAttrs.entrySet().toArray(newAttrArray(childAttrs.size()));
    }

    p.addLast(new ChannelInitializer<Channel>() {
        @Override
        public void initChannel(Channel ch) throws Exception {
            final ChannelPipeline pipeline = ch.pipeline();
            ChannelHandler handler = config.handler();
            if (handler != null) {
                pipeline.addLast(handler);
            }

            // 通过子线程把ServerBootstrapAcceptor添加到pipeline中
            ch.eventLoop().execute(new Runnable() {
                @Override
                public void run() {
                    pipeline.addLast(new ServerBootstrapAcceptor(
                            currentChildGroup, currentChildHandler, currentChildOptions, currentChildAttrs));
                }
            });
        }
    });
}
```

可以看到在init方法中除了给channel配置参数之外，还对channel绑定的pipeline做了一些配置。对pipeline的配置通过一个特殊的`ChannelInitializer`实现，后者继承了`ChannelInboundHandlerAdapter()`，主要作用就是用来初始化pipeline，它重写了`channelRegistered`方法(连接首先触发此方法)回调`initChannel(channel)`方法，用户在此方法内通过`addLast`方法定义pipeline内的handler结构。为了防止反复触发pipeline的修改，在一次调用ChannelInitializer后会移除它自己。

回到init方法中的ChannelInitializer实现，它先把配置中的handler添加到pipeline中(本例中ServerBootstrap未定义handler故为null)，然后通过子线程异步地往pipeline中添加一个`ServerBootstrapAcceptor`。由于当前channel是一个nettyServerSocketChannel，自然它将接收到的请求都是连接类型的。所以连接进来之后，都会经过`ServerBootstrapAcceptor`的处理，它是连接建立的核心。



## ServerBootstrapAcceptor

这是ServerBootstrap中定义的一个内部类，继承了`ChannelInboundHandlerAdapter`，重写了`channelRead()`方法。对于ServerSocketChannel，channelRead是连接建立后回调的方法：

```java
// ServerBootstrap$ChannelInboundHandlerAdapter.java

public void channelRead(ChannelHandlerContext ctx, Object msg) {
    // 对于新建连接, 此channel为NioSocketChannel(netty对java SocketChannel的封装)
    final Channel child = (Channel) msg;

    // 给子Channel即NioSocketChannel添加配置的childHandler, 在本例中是在EchoServer面方法中定义
    // 的ChannelInitializer
    child.pipeline().addLast(childHandler);

    // 配置channel... (忽略代码)

    try {
        // 将此SocketChannel注册到childGroup(workerGroup)，即将此channel添加到worker线程池中处理
        childGroup.register(child).addListener(new ChannelFutureListener() {
            @Override
            public void operationComplete(ChannelFuture future) throws Exception {
                if (!future.isSuccess()) {
                    forceClose(child, future.cause()); // 异常关闭
                }
            }
        });
    } catch (Throwable t) {
        forceClose(child, t); // 异常关闭
    }
}
```

以上代码首先做的是将`ServerBootstrap#childHandler()`方法配置的handler添加到pipeline中，回看EchoServer配置的也是ChannelInitializer。重点在方法最后通过调用`childGroup#register(Channel)`把channel注册到childGroup中。这是NioSocketChannel从BossGroup到WorkerGroup转移的关键一步。

`childGroup`在EchoServer的例子中实现为`NioEventLoopGroup`，调用register方法实际调用的是其父类`MultithreadEventLoopGroup`中的实现：

```java
// MultithreadEventLoopGroup.java
public ChannelFuture register(Channel channel) {
    // 将channel注册到next()方法从Group中选择的一个线程(EventLoop)中
    // next()方法使用 DefaultEventExecutorChooserFactory 提供的实现
    return next().register(channel);
}
```

在next()方法从线程池中选择了一个EventLoop(线程,SingleThreadEventLoop)后，调用后者的register(channel)方法

```java
# SingleThreadEventLoop.java
@Override
public ChannelFuture register(Channel channel) {
    return register(new DefaultChannelPromise(channel, this));
}

@Override
public ChannelFuture register(final ChannelPromise promise) {
    ObjectUtil.checkNotNull(promise, "promise");
    // 调用底层register方法(异步调用), 并将结果写入promise
    promise.channel().unsafe().register(this, promise);
    // 直接返回promise
    return promise;
}

# AbstractChannel.java
private void register0(ChannelPromise promise) {
    try {
        boolean firstRegistration = neverRegistered;
        doRegister(); // 核心调用1. 在AbstractNioChannel的实现中, 将channel注册到selector上, 但实际未监听任意事件
        neverRegistered = false;
        registered = true;

        safeSetSuccess(promise); // 更新promise状态为SUCCESS
        // 在pipeline上触发一个ChannelRegistered事件
        pipeline.fireChannelRegistered();
        if (isActive()) { // 判断channel是否已经完成连接
            if (firstRegistration) {
                // 首次连接, 以ChannelActive事件从head开始触发Pipeline
                pipeline.fireChannelActive(); 核心调用2.
            } else if (config().isAutoRead()) {
                // 不回调channelActive(), 直接回调beginRead()方法注册OP_XXX事件
                beginRead();
            }
        }
    } catch (Throwable t) {
        // Close the channel directly to avoid FD leak.
        closeForcibly();
        closeFuture.setClosed();
        safeSetFailure(promise, t);
    }
}
```

调用register方法后会经过一系列的状态校验，但最终的核心操作落在两个地方：

**核心调用1. doRegister().** 
此方法由`AbstractNioChannel`实现，尝试调用`nioChannel#register()`方法将当前`niochannel`注册到所属工作线程的Selector上，但需注意此时未注册监听任意事件(ops=0)。

```java
# AbstractNioChannel.java
protected void doRegister() throws Exception {
    for (;;) {
        try {
            logger.info("AbstractNioChannel register 0");
            // 将channel注册到selector上, 但是这里ops为0未监听任何事件
            selectionKey = javaChannel().register(eventLoop().selector, 0, this);
            return;
        } catch (CancelledKeyException e) {
            // 异常处理...
        }
    }
}
```

**核心调用2. fireChannelActive()**

fireChannelActive方法的作用是按顺序回调Pipeline链上的所有有关Handler的`channelActive`方法。前面提过piepeline默认的实现为`DefaultChannelPipeline`，而这个Pipeline自带了两个Handler:head和tail。Server对于Client发来的建立连接请求，显然是入站事件，所以会从head开始依次调用InboundHandler。

```
# DefaultChannelPipeline$HeadContext.java
public void channelActive(ChannelHandlerContext ctx) throws Exception {
    ctx.fireChannelActive(); // 调用Pipeline链上的handler回调方法
    readIfIsAutoRead(); // 注册OP_XXX事件
}
```

HeadContext的`channelActive`方法非常简单，第一行链式回调hander方法；第二行则判断`autoRead`参数为true进而开始消息读取。这里的读取是通过`pipeline#read()`方法开始的，从pipeline的tail到head，依次调用`OutboundHandler#read()`方法，最终同样是落到HeadContext中的read方法:

```
# DefaultChannelPipeline$HeadContext.java
public void read(ChannelHandlerContext ctx) {
    unsafe.beginRead();
}

# AbstractChannel$AbstractUnsafe.java
public final void beginRead() {
    assertEventLoop();

    if (!isActive()) {
        return;
    }
    try {
        doBeginRead();
    } catch (final Exception e) {
        invokeLater(new Runnable() {
            @Override
            public void run() {
                pipeline.fireExceptionCaught(e);
            }
        });
        close(voidPromise());
    }
}

# AbstractNioChannel
protected void doBeginRead() throws Exception {
    // Channel.read() or ChannelHandlerContext.read() was called
    final SelectionKey selectionKey = this.selectionKey;
    if (!selectionKey.isValid()) {
        return;
    }

    readPending = true;

    final int interestOps = selectionKey.interestOps();
    if ((interestOps & readInterestOp) == 0) {
        // readInterestOp是AbstractNioChannel的实例属性,在创建Channel的
        // 时候创建
        selectionKey.interestOps(interestOps | readInterestOp);
    }
}
```

可以看到最后由`AbstractChannel#doBeginRead()`方法执行SelectionKey的注册，至于注册的事件是什么，由创建AbstractChannel实例时构造方法指定。对于NioServerSocketChannel，这个值为OP_ACCEPT；对于NioSocketChannel这个值为OP_READ。

## connect(HOST,PORT)

connect(HOST,PORT)方法是客户端以`NioSocketChannel#connect()`方法封装的调用过程，跟Server端的bind()方法比较类似，这里不再赘述。留给你自己去跟踪源码阅读。

# 线程启动

细心的你发现，在EchoServer/EchoClient中，没有显示启动线程池的方法，那么在什么地方做了这样的活？答案是在`ServerBootstrap#bind()` 和 `Bootstrap#connect()`。在这两个方法调用的最后分别落到`AbstractBootstrap#doBind0()`和`Bootstrap#doConnect()`方法。这两个方法中以`Executor#execute(Runnable)`的方式异步提交了执行channel注册&连接的方法。也就是在execute方法内部启动了线程。

```
# AbstractBootstrap.java
private static void doBind0(
        final ChannelFuture regFuture, final Channel channel,
        final SocketAddress localAddress, final ChannelPromise promise) {

    // This method is invoked before channelRegistered() is triggered.  Give user handlers a chance to set up
    // the pipeline in its channelRegistered() implementation.
    channel.eventLoop().execute(new Runnable() {
        @Override
        public void run() {
            if (regFuture.isSuccess()) {
                channel.bind(localAddress, promise).addListener(ChannelFutureListener.CLOSE_ON_FAILURE);
            } else {
                promise.setFailure(regFuture.cause());
            }
        }
    });
}

# Bootstrap.java
private static void doConnect(
        final SocketAddress remoteAddress, final SocketAddress localAddress, final ChannelPromise connectPromise) {

    // This method is invoked before channelRegistered() is triggered.  Give user handlers a chance to set up
    // the pipeline in its channelRegistered() implementation.
    final Channel channel = connectPromise.channel();
    channel.eventLoop().execute(new Runnable() {
        @Override
        public void run() {
            if (localAddress == null) {
                channel.connect(remoteAddress, connectPromise);
            } else {
                channel.connect(remoteAddress, localAddress, connectPromise);
            }
            connectPromise.addListener(ChannelFutureListener.CLOSE_ON_FAILURE);
        }
    });
}
```



# 小结

本文我带你分析了Netty的Reactor线程模型，这是netty处理一切任务的基础，server端分为bossGroup和workerGroup两个线程池，分别处理连接请求 和 IO请求，client端只有workerGroup一个线程池。这跟server端由ServerSocketChannel和SocketChannel，而客户端只有SocketChannel是某种意义上的对应。通过调整两个线程池的配置，可以调整连接和IO的处理能力。

另外本文重点分析了Server端创建连接的完整过程。连接的本质是ServerSocketChannel#bind和SocketChannel#connect，在netty中把这两个方法做了AbstractChannel, AbstractNioChannel, NioServerSocketChannel的几层抽象，最底层还设计了一个Unsafe接口用于直接操作底层网络操作，这有效地屏蔽了各种通信方式的细节。

![img](http://minio.gogodjzhu.com/images/20210404_232959_fc35324f-1a2c-426f-ae87-f1a394bd820a.png)

最后需要提醒你关注的是netty对Pipeline+Handler的操作模式，这是贯穿连接，读写，业务功能等的整体设计，本文做了简单的介绍，后文还会继续讨论，非常值得你仔细研究。