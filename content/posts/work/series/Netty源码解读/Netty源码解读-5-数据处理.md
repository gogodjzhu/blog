---
title: "Netty源码解读(5)-数据处理(Handler调用链)"
date: 2020-02-01T18:22:44+08:00
description: "Netty框架将网络处理的场景抽象为一系列责任链模式设计的Handler，基于Netty实现业务逻辑的本质是编排Handler和重写对应的Handler方法。本章我们就来研究下这个Netty设计的精髓。"
tags: ["网络", "中间件", "JAVA", "Netty"]
categories: "Netty源码解读"
draft: false
---

上一章介绍了Netty如何基于`ByteBuf`从channel中读取数据：

1. 每次读取完成之后可以得到一个填充了业务数据的`ByteBuf`，并作为参数调用`pipeline#fireChannelRead(ByteBuf)`方法
2. 读取完成之后(可能是数据读完/或者达到连续读的最大值控制/或者连接断开导致)会触发一次`pipeline#fireChannelReadComplete()`方法

`Pipeline`是Netty实现业务逻辑的核心组件，定义为`ChannelPipeline`接口，每个Channel创建的时候都会绑定一个Pipeline，这步操作是在Channel的抽象父类`AbstractChannel`的构造方法中实现的：

```java
# AbstractChannel.java
protected AbstractChannel(Channel parent) {
    this.parent = parent;
    id = newId();
    unsafe = newUnsafe();
    pipeline = newChannelPipeline(); // 创建并绑定当前channel实例的pipeline
}

protected DefaultChannelPipeline newChannelPipeline() {
    return new DefaultChannelPipeline(this);
}
```

即，ChannelPipeline默认使用的是`DefaultChannelPipeline`。

# ChannelPipeline的基础流程

`DefaultChannelPipeline`在本质上是一个双向链表结构，内部的节点通过`AbstractChannelHandlerContext`实现，后者拥有两个指针属性用于记录在pipeline链上的前后节点：

```java
# AbstractChannelHandlerContext.java
volatile AbstractChannelHandlerContext next;
volatile AbstractChannelHandlerContext prev;
```

而Pipeline自己保存了整个链的首尾指针：

```java
# DefaultChannelPipeline.java
final AbstractChannelHandlerContext head;
final AbstractChannelHandlerContext tail;
```

所有的出入站事件都是从`head`或者`tail`开始触发，并依赖`AbstractChannelHandlerContext`的双指针进行传播的。

- Handler调用链

  ![img](http://minio.gogodjzhu.com/images/20210418_161049_c4b46104-b39c-45b8-9893-27e158a98b49.png)



还是回到本章开头的内容，当一个读事件触发的时，调用了`pipeline#fireChannelRead()`方法：

```java
# DefaultChannelPipeline.java
public final ChannelPipeline fireChannelRead(Object msg) {
    // 由于读入消息的处理显然是一个入站事件，所以是从head开始调用整个Pipeline链的
    AbstractChannelHandlerContext.invokeChannelRead(head, msg);
    return this;
}

# AbstractChannelHandlerContext.java
static void invokeChannelRead(final AbstractChannelHandlerContext next, Object msg) {
    // 非空判断
    final Object m = next.pipeline.touch(ObjectUtil.checkNotNull(msg, "msg"), next);
    // 获取context绑定的Executor(即EventLoop或者EventLoopGroup)
    EventExecutor executor = next.executor();
    // 判断当前线程即是EventLoop,那么在当前线程直接执行方法(想让与直接执行Runnable#run方法)
    if (executor.inEventLoop()) {
        next.invokeChannelRead(m);
    } else {
        // 当前线程不是EventLoop, 那么需要将待执行方法封装成一个Runnable, 提交给Executor来异步线程执行
        executor.execute(new Runnable() {
            @Override
            public void run() {
                next.invokeChannelRead(m);
            }
        });
    }
}
```



不论在当前线程直接执行还是通过`Executor#execute()`方法另起线程执行，最终调用的都是同一个方法：

```java
# AbstractChannelHandlerContext.java
private void invokeChannelRead(Object msg) {
    // 判断Handler是否处于可执行状态
    if (invokeHandler()) {
        try {
            ((ChannelInboundHandler) handler()).channelRead(this, msg);
        } catch (Throwable t) {
            notifyHandlerException(t);
        }
    } else {
        // 跳过当前Handler的处理, 直接传播给下级
        fireChannelRead(msg);
    }
}
```

这里存在一个Handler状态判断的处理`invokeHandler()`，后文再做讨论，这里先告诉你对于`HeadContext`此时在第一个if判断为true。所以进入当前`HeadContext#channelRead()`方法的执行：

```java
# DefaultChannelPipeline$HeadContext.java
public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
    // 注意这里是context.fireChannelRead, 而不是pipeline的同名方法
    // HeadContext未重写此方法, 这里调用父类AbstractChannelHandlerContext的实现
    ctx.fireChannelRead(msg);
}

# AbstractChannelHandlerContext.java
@Override
public ChannelHandlerContext fireChannelRead(final Object msg) {
    invokeChannelRead(findContextInbound(), msg);
    return this;
}

# 在pipelinet链表上获取正序的下一个Inbound类型Context, 如果非Inbound会跳过
private AbstractChannelHandlerContext findContextInbound() {
    AbstractChannelHandlerContext ctx = this; // 从当前context开始遍历
    do {
        ctx = ctx.next;
    } while (!ctx.inbound); // 跳过所有非Inbound的handler
    return ctx;
}
```

上面的几段代码是链表调用，在有限的几个方法内来回调用，初看可能会有点绕，建议打开断点，跟踪调用栈会帮助你更快地了解这里的底层逻辑。

简单小结Pipeline实现链上Handler顺序调用的调用顺序：

1. 通过`OP_READ`事件触发`NioEventLoop`读入`ByteBuf`后，调用`Pipeline#fireChannelRead(ByteBuf)`方法开始
2. `Pipeline`的默认实现`DefaultChannelPipeline`写死了入站事件的处理逻辑是从`head`开始的。即`invokeChannelRead(head, byteBuf)`。
3. `AbstractChannelHandlerContext#invokeChannelRead(context, byteBuf)`方法本质是调用context绑定handler的`channelRead(ctx:ChannelHandlerContext, msg:Object)`方法。
4. 一旦触发某个handler的`channelRead`方法后，在pipeline上的调用链便断开，若想继续在pipeline上传递事件，需要调用context的`fireChannelRead(msg:Object)`方法。`HeadContext`则是未作任何处理直接调用`fireChannelRead()`方法

# 几个细节

数据处理的大体流程就是这样，下面来讨论几个细节：

- 执行Handler方法的线程是怎么决定的？同一个连接发来的多个请求，是否落在同一个线程中执行？
- 已知我们可以给单个Handler指定专属的线程池，怎么实现？
- 在多个线程(线程池)之间如何做同步？

## Handler处理线程的选择&启动

当一个新的连接事件进来之后，`bossWorkerGroup`会监听到新的`SelectionKey#OP_ACCEPT`。这个方法在`NioEventLoop#processSelectedKey`方法内：

```java
# NioEventLoop.processSelectedKey(k:SelectionKey, ch AbsNioChannel)

final AbstractNioChannel.NioUnsafe unsafe = ch.unsafe();

if ((readyOps & (SelectionKey.OP_READ | SelectionKey.OP_ACCEPT)) != 0 || readyOps == 0) {
    unsafe.read();
    if (!ch.isOpen()) {
        // Connection already closed - no need to handle write.
        return;
    }
}
```

可见NioEventLoop线程把`OP_READ`和`OP_ACCEPT`都是交给channel绑定的unsafe类来处理。接收到`OP_ACCEPT`事件的显然是`ServerSocketChannel`，而处理此事件的Handler在HeadContext之后，是`ServerBootstrapAcceptor`。

```java
# ServerBootstrap$SererBootstrapAcceptor.java
public void channelRead(ChannelHandlerContext ctx, Object msg) {
    // 对于新建连接, 此channel为io.netty.channel.socket.nio.NioSocketChannel(netty对java SocketChannel的封装)
    final Channel child = (Channel) msg;

    // 将ServerBootstrap#childHandler()方法配置的Handler添加进来
    child.pipeline().addLast(childHandler);

    // (省略) 利用config配置childChannel...

    try {
       // 将此SocketChannel注册到childGroup(workerGroup), 并在判断执行线
       // 程非Channel绑定的workerNioEventLoop时,启动worker线程执行具体
       // 的channel绑定任务
        childGroup.register(child).addListener(new ChannelFutureListener() {
            @Override
            public void operationComplete(ChannelFuture future) throws Exception {
                if (!future.isSuccess()) {
                    forceClose(child, future.cause());
                }
            }
        });
    } catch (Throwable t) {
        forceClose(child, t);
    }
}

# SingleThreadEventLoop.java
public ChannelFuture register(Channel channel) {
    return register(new DefaultChannelPromise(channel, this));
}

# SingleThreadEventLoop.java
public ChannelFuture register(final ChannelPromise promise) {
    ObjectUtil.checkNotNull(promise, "promise");
    // 如果当前线程不是EventLoop线程, 会把register工作以Task的方式交给EventLoop异步处理, 处理结果会写入promise中
    promise.channel().unsafe().register(this, promise);
    // 直接返回promise
    return promise;
}

# AbstractChannel$AbstractUnsafe.java
public final void register(EventLoop eventLoop, final ChannelPromise promise) {
    // (省略) eventLoop状态校验
    AbstractChannel.this.eventLoop = eventLoop;

    if (eventLoop.inEventLoop()) {
        register0(promise);
    } else {
        // 判断当前线程非EventLoop实体所属的线程, 启动之
        try {
            eventLoop.execute(new Runnable() {
                @Override
                public void run() {
                    register0(promise);
                }
            });
        } catch (Throwable t) {
            // (省略) 异常处理
        }
    }
}
```

在上述代码端的最后一个`register()`方法中，完成了从`bossWorkerGroup`到`workerWorkerGroup`的切换。在调用`evnetLoop#execute(Runnable)`方法的内部，启动了`worker`线程，也就是这个`eventLoop`线程实现。从这之后，`worker`线程接受到的读写请求都在此线程内执行，亦即处理的Handler在此线程内执行。

## Handler单独配置EventLoop

除了`bossWorkerGroup`接收新连接、创建pipeline时给后者绑定的EventLoop之外，我们还可以给piepeline中的Handler单独配置执行线程(池)。只需要稍微修改Pipeline添加Handler的方法调用即可，如下：

```java
pipeline.addLast(new NioEventLoopGroup(3), serverHandler)
```

那么另一个问题马上接踵而来，以上面这行代码为例，我们给一个`Handler`指定了一个拥有三个线程的线程池，那么每次调用Handler时，使用哪个线程呢？

要回答这个问题，得回到Pipeline的构造中去找答案。以EchoServer为例，通过`ServerBootstrap#childHandler(ChannelInitializer)`方法完成了Pipeline的配置，其中就包括调用addLast()方法添加Handler。而handler是通过`DefaultChannelPipeline#newContext(EventExecutorGroup, contextname: String, ChannelHandler)`方法封装成Context后才添加到Pipeline的。

```java
# DefaultChannelPipeline.java

private AbstractChannelHandlerContext newContext(EventExecutorGroup group, String name, ChannelHandler handler) {
    return new DefaultChannelHandlerContext(this, childExecutor(group), name, handler);
}

/**
* 根据Group选择Executor
**/
private EventExecutor childExecutor(EventExecutorGroup group) {
    // 若不指定, 则不使用子Executor
    if (group == null) {
        return null;
    }
    // 配置项，默认true. 控制同一个Pipeline下的所有Handler是否应该使用同一个EventExecutor
    Boolean pinEventExecutor = channel.config().getOption(ChannelOption.SINGLE_EVENTEXECUTOR_PER_GROUP);
    if (pinEventExecutor != null && !pinEventExecutor) {
        return group.next();
    }
    Map<EventExecutorGroup, EventExecutor> childExecutors = this.childExecutors;
    if (childExecutors == null) {
        // netty的作者认为正常情况下不会使用超过4个线程池, 所以初始容量设为4
        childExecutors = this.childExecutors = new IdentityHashMap<EventExecutorGroup, EventExecutor>(4);
    }
    // Pin one of the child executors once and remember it so that the same child executor
    // is used to fire events for the same channel.
    EventExecutor childExecutor = childExecutors.get(group);
    if (childExecutor == null) {
        childExecutor = group.next();
        childExecutors.put(group, childExecutor);
    }
    return childExecutor;
}
```

核心的方法在`childExecutor(EventExecutorGroup)`方法，它通过一个`Map<Group, Executor>`控制在同一个业务线程池内，分配给同一个ChannelPipeline的所有Handler的总是同一个Executor，即同一个线程处理一个Pipeline实例下的所有Handler。



## 线程池之间的同步

至今为止，我们讨论了Netty中的三种线程池：

- BossGroup 负责处理TCP连接
- WorkerGroup 负责处理IO
- handlerGroup 针对Handler进行配置，配置了此线程池的handler只会从此池固定分配一个Executor（即一个固定的线程）

由于Netty采用的`Reactor线程模型`，比起`共享内存的线程模型`要更容易实现线程同步。一个线程到另一个线程的数据传输应该尽可能使用消息传递的方式来实现。但也有特殊情况，那就是共享Handler实例。

需要在多个Pipeline之间共享的Handler，首先必须添加`@Sharable`注解，然后需要开发这自行保证Handler内方法的线程安全。

其次可以关注`AbstractChannelHandlerContext`类中有一个实例属性`executor:EventExecutor`，即此Context（Handler）绑定的子执行器。若不为空，则使用这此executor来处理Handler方法；若为空则取Pipeline绑定的`EventLoop`。

executor的值根据addLast方法来确定，若通过不带`EventLoopGroup`参数的`Pipeline#addLast()`方法添加Handler，那么此handler所属的Context的executor为null, 否则会从指定的Group中选择一个线程来执行(具体逻辑参看前文)。



# 总结

至此我给你介绍了Netty基于Handler调用链的业务处理逻辑。

首先介绍了Handler调用链的实现基础，也即`Pipeline`，是一个双向链表结构，链上的每个节点为`HandlerContext`。

然后我给你分析了Pipeline的默认实现`DefaultChannelPipeline`，在接受到新的可读数据时，会从头节点`HeadContext`开始遍历整个链表进行调用。遍历的过程中会跳过未实现Inbound（或者Outbound，根据出入站事件而定）的Handler。而数据要在Pipeline上连续调用，必须在Handler上通过`Context#fireChannelRead(Object)`来触发下一个Handler来传播。

最后我给你介绍了Netty如何分配执行线程，自定义Handler线程的方法，以及现线程同步的原理。