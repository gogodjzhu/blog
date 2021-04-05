---
title: "Java线程池源码解析"
date: 2019-09-03T21:11:13+08:00
description: "并发编程中常用到线程池，你是否还停留在fix，cache，single几种线程池的简单使用? 是时候深入一下源代码了"
tags: ["JAVA", "并发编程"]
categories: "Java原理系列"
draft: false
---

讲线程池， 首先简单看看Java中单个线程的实现方式



# Thread & Runnable & Callable

这3个接口是所有多线程开发的基础。 Runnable和Callable是对`任务`的抽象， 分别代表了有返回的和无返回的任务类型。各自只定义一个方法， 就是这个`任务`该做什么

```java
public interface Runnable {
    public abstract void run();
}
public interface Callable<V> {
    V call() throws Exception;
}
```

有了`任务`之后， 我们当然可以让`main`线程来处理它， 但是在设计上， 这两个接口是为了让子线程调用做的。 怎样启动子线程来执行任务? 这就需要用上`Thread`类， 相信大家再熟悉不过， 它实现了`Runnable`接口， 内部定义了如何启动新线程的native方法`start0`(封装在`start`方法内)， `start0`启动了子线程后会在子线程内回调`run`方法， 从而实现子线程执行目标`任务`。 

native方法是如何启动新线程， 怎样跟操作系统线程映射的， 我们这里咱不做深入。 

由于启动一个新的线程必须通过`Thread`， 调用native方法， 触及到操作系统层面的开销往往很大。 通过线程池， 可以缓存部分线程， 复用已有线程， 避免频繁调用native方法， 是最主要的目标。

> **线程池 vs 连接池**
>
> 通过池化复用资源是开发中常见的性能优化方案，我们通常面对两类池: 线程池和连接池。 但虽然名字都叫池， 但在底层各自讨论的其实不是一个概念。
>
> 对于线程池， 池化的对象是线程， 线程的本质是CPU的运行时间片， 池化线程， 其实是提前申请一批CPU时间片， 在时间片到来时， 直接处理任务即可。 好处是不需要每次处理任务的时候再单独申请。
>
> 对于连接池， 池化的对象是连接， 而连接的本质是一个句柄(引用)， 它是经过协议握手之后保留下来的可直接使用的执行入口。 池化连接， 可以避免频繁的协议握手操作。



# 手写线程池

思考一下， 如果让我们自己实现一个线程池， 我们会怎么做? 下面是一个简单的demo。

```java
public class SimpleThreadPool {

    // 线程池关闭标志
    private boolean shutdown;

    // worker线程运行信号
    private CountDownLatch countDownLatch;

    // 任务队列
    private ConcurrentLinkedQueue<Runnable> taskQueue
            = new ConcurrentLinkedQueue<>();

    public SimpleThreadPool(int threadNum) {
        countDownLatch = new CountDownLatch(threadNum);
        for (int i = 0; i < threadNum; i++) {
            // 定义并启动工作线程
            new Thread(new Runnable() {
                @Override
                public void run() {
                    try {
                        while (!taskQueue.isEmpty() || !shutdown) {
                            Runnable r = taskQueue.poll();
                            if (r == null) {
                                Thread.sleep(100);
                                continue;
                            }
                            r.run();
                        }
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    } finally {
                        countDownLatch.countDown();
                    }
                }
            }).start();
        }
    }

    public void submit(Runnable runnable) {
        taskQueue.add(runnable);
    }

    public void shutdown() throws InterruptedException {
        shutdown = true; // 关闭线程池标志
        countDownLatch.await(); // 等待worker线程退出
    }

}

public class SimpleThreadPoolDemo {

    public static void main(String[] args) 
            throws InterruptedException {
        SimpleThreadPool simpleThreadPool = new SimpleThreadPool(2);
        class PrintRunnable implements Runnable{
            @Override
            public void run() {
                System.out.println("current thread:" + 
                        Thread.currentThread().getId());
            }
        }
        for (int i = 0; i < 10; i++) {
            simpleThreadPool.submit(new PrintRunnable());
        }
        simpleThreadPool.shutdown();
    }

}
```

麻雀虽小， 五脏却全， `SimpleThreadPool`已经包含了线程池所需要的几大要素， 并且在实际开发中常用的`ThreadPoolExecutor`线程池里也有1比1的对应定义: 

- **运行时状态**，`SimpleThreadPool`只有`shutdown`关闭标记，`workQueue`保存当前的工作线程。 而`ThreadPoolExecutor`提供了更加丰富的运行时状态， 更好地监控线程池运行状态的同时， 也可以对线程池实施更细粒度的控制。
- **任务队列**，(`SimpleThreadPool.taskQueue` 对应 `ThreadPoolExecutor.workQueue`) 用于保存待执行的任务。
- **工作线程**， 在`SimpleThreadPool`中采用的是固定数量的线程， 每个工作线程并发地从任务队列中获取任务执行， 而`ThreadPoolExecutor`支持丰富的自定义方案， 能够根据配置动态调整工作线程的数量。 
- **并发控制**， 这个在我们的`SimpleThreadPool`中做的很简陋(甚至难免会有bug)， 而`ThreadPoolExecutor`则使用更加精妙的方法实现并发安全控制。



# ThreadPoolExecutor

行文至此， 本文的主角终于登场。`ThreadPoolExecutor`是常用的线程池实现类， 可能开发中并不会直接配置它（但是阿里巴巴的开发规约推荐你了解并直接使用），而是使用更加方便的`Executors`创建线程池。 其实后者创建不同类型（single，fixed，cached）的线程池，无非是使用不同的参数创建`ThreadPoolExecutor`实例而已。



## 继承关系 & 属性

下面是`ThreadPoolExecutor`的类继承图， 可以清晰地看出功能继承关系：

![img](http://minio.gogodjzhu.com/images/20210404_172039_a7573641-b2a0-413d-9aad-c6359c9fc627.png)

`Executor`是顶层接口，它是处理不带返回的任务(`Runnable`)的基础执行器。`ExecutorService`则进一步定义了`ThreadPoolExecutor`线程池最基础的一些方法，包括:

1. **关闭方法**(`shutdown()`和`shutdownNow()`)，区别在如何对待正在执行/排队的任务，是关闭它，还是等待执行完成

2. **提交单个任务的方法**(`submit(Runnable, T) : Future <T>`， `submit(Callable<T>) : Future<T>`)， 和**提交批量任务的方法**`invokeAll(tasks : Collection<? extends Callable<T>>) : List<Future<T>>`。这些方法返回的都是一个(批量是多个)`Future`对象。

   >  在`ThreadPoolExecuter`中返回的是实现类`FutureTask`， 支持异步对任务进行 执行/等待/停止 操作。

`AbstractExecutorService`抽象类实现了核心的`submit`，`invokeAny`， `invokeAll`方法， 实现的本质都是将Runnable/Callable封装成`FutureTask`对象， 调用抽象方法`execute`(异步方法， 不阻塞)， 最后返回`FutureTask`对象给调用方。

```java
// AbstractExecutorService
public Future<?> submit(Runnable task) {
    if (task == null) throw new NullPointerException();
    // 调用newTaskFor封装成FutureTask
    RunnableFuture<Void> ftask = newTaskFor(task， null);
    execute(ftask); // 这是子类实现各自逻辑的入口方法，ThreadPoolFactory中的实现看后文
    return ftask;
}
```

下图是ThreadPoolExecutor定义的所有属性:

![img](http://minio.gogodjzhu.com/images/20210404_172056_e3f43b29-3a5b-4e92-ba5d-a33988514820.png)

基本上都见名知意， 这里摘几个重要的赘述一下:

- `corePoolSize`，`maximumPoolSize`，`largestPoolSize`
  这几个是线程池大小的核心配置， `corePoolSize`决定了核心线程数， 核心线程即便在没有任务的时候也不会回收。 `maximumPoolSize`则是允许最大的线程数， 当任务数大于`corePoolSize`时， 会临时增加工作线程。 超过`corePollSize`小于`maximumPoolSize`的线程会在闲置一段时间后被回收。 `largestPoolSize`则是线程池实例达到过的最高工作线程数， 利用它， 结合`workers`实际大小可以对线程池的指标进行监控和调优。
- `workQueue`，`workers`
  `workQueue`是任务队列， `Executor`底层默认通过`LinkedBlockingQueue`实现。
  `workers`是工作线程集合
- `allowCoreThreadTimeOut`，`keepAliveTime`
  当`allowCoreThreadTimeOut`为`true`时， 核心线程空闲时间超过`keepAliveTime`也会被回收
- `threadFactory`
  线程工厂，当需要新增工作线程的时候通过它来创建。`Executors`默认使用内部类`DefaultThreadFactory`的实现。

以上几个就是线程池最核心的几个配置，`Executors`创建线程池修改的主要就是这几个(如果不是全部的话)参数。 下面就来看看依赖这些参数， 线程池是如何运行的。



## 运行时配置

### 启动线程池

`ThreadPoolExecutor`源码中是没有start相关方法的， 构造方法所做的也只是初始化一些对象属性

```java
public ThreadPoolExecutor(int corePoolSize，
                          int maximumPoolSize，
                          long keepAliveTime，
                          TimeUnit unit，
                          BlockingQueue<Runnable> workQueue，
                          ThreadFactory threadFactory，
                          RejectedExecutionHandler handler) {
    // 参数合法性校验
    if (corePoolSize < 0 ||
        maximumPoolSize <= 0 ||
        maximumPoolSize < corePoolSize ||
        keepAliveTime < 0)
        throw new IllegalArgumentException();
    if (workQueue == null || threadFactory == null || handler == null)
        throw new NullPointerException();
    // 赋初值
    this.corePoolSize = corePoolSize;
    this.maximumPoolSize = maximumPoolSize;
    this.workQueue = workQueue;
    this.keepAliveTime = unit.toNanos(keepAliveTime);
    this.threadFactory = threadFactory;
    this.handler = handler;
}
```

除此之外，` ThreadPoolExecutor`还定义了运行状态变量`ctl`， 这是一个int类型(32bit长度)， 高3位保存运行状态， 低位用来保存运行中的工作线程数。 所以线程池最大能够支撑的线程数为`2^29-1`。 

```java
private final AtomicInteger ctl = new AtomicInteger(ctlOf(RUNNING， 0));

private static final int COUNT_BITS = Integer.SIZE - 3;
private static final int CAPACITY   = (1 << COUNT_BITS) - 1;

// 运行中
private static final int RUNNING    = -1 << COUNT_BITS;
// 已关闭， 工作线程还在运行， 只是不接收新任务
private static final int SHUTDOWN   =  0 << COUNT_BITS;
// 已关闭， 强制中断运行中的工作线程， 同样不接收新任务
private static final int STOP       =  1 << COUNT_BITS;
// 所有的工作线程已停止， 等待执行terminated方法(默认为空方法， 给子类实现功能)
private static final int TIDYING    =  2 << COUNT_BITS;
// 已执行terminated方法
private static final int TERMINATED =  3 << COUNT_BITS;

// 封装和解析ctl
private static int runStateOf(int c)     { return c & ~CAPACITY; }
private static int workerCountOf(int c)  { return c & CAPACITY; }
private static int ctlOf(int rs， int wc) { return rs | wc; }
```



### 添加任务 & 增加工作线程

创建线程池对象之后，并无start方法来启动它，此时并没有工作线程运行。要让线程池工作起来须添加任务，对应的方法是`AbstractExecutorService.sumbit`(前文已经提过)。`submit`方法会生成一个`FutureTask`对象， 正是在调用它实现的`execute(Future)`方法内部完成了启动工作线程的工作。

```java
// ThreadPoolExecutor.java
public void execute(Runnable command) {
    if (command == null)
        throw new NullPointerException();
    // 线程池运行状态 
    int c = ctl.get();
    // 工作线程数小于核心线程数， 添加一个新工作线程， 并将command作为第一个任务
    if (workerCountOf(c) < corePoolSize) {
        if (addWorker(command, true))
            return;
        c = ctl.get();
    }
    // 工作线程新增失败时， 将任务添加到任务队列
    if (isRunning(c) && workQueue.offer(command)) {
        // 由于方法未加锁， 所以做二次判断， 保证线程池关闭时正确回调reject方法， 
        // 线程池运行时至少有一个工作线程
        int recheck = ctl.get();
        if (!isRunning(recheck) && remove(command))
            reject(command);
        else if (workerCountOf(recheck) == 0)
            addWorker(null, false);
    }
    // 添加至队列失败，回调RejectedExecutionHandler，默认的实现是抛出对应异常
    else if (!addWorker(command, false))
        reject(command);
}
```

`addWorker()`方法所做的，是在一系列的状态检查(包括线程数限制，线程池运行状态判断等等)通过之后将`Runnable`封装成Worker对象，添加到`workers`工作线程集合，最后启动它。也就是说，真正的工作线程是被封装在Worker对象中的。Worker对象继承自AQS类， 以支持加锁操作；实现了Runnable接口，`run`方法是子线程启动的入口。 

```java
// ThreadPoolExecutor内部类
private final class Worker
    extends AbstractQueuedSynchronizer
    implements Runnable{

    final Thread thread;
    Runnable firstTask;
    volatile long completedTasks; // 工作线程完成的任务数，失败的也计算在内

    Worker(Runnable firstTask) {
        setState(-1); // 设置父类AQS的state，用于防止错误中断
        this.firstTask = firstTask;
        // 这里是关键!! 
        // 通过ThreadFactory来创建新的线程，入参为this(implements Runnable)，所以外层调用会通过this.thread.start()启动此线程，子线程内实际执行的方法是run()
        this.thread = getThreadFactory().newThread(this);
    }

    // 工作线程的核心操作在runWorker中
    public void run() {
        runWorker(this);
    }

    // ...
    
    // 注意本方法是由子线程的run()方法调用的
    final void runWorker(Worker w) {
        Thread wt = Thread.currentThread();
        Runnable task = w.firstTask; // 创建Worker对象的时候带进来的初始Task
        w.firstTask = null;
        w.unlock(); // allow interrupts
        boolean completedAbruptly = true;
        try {
            // 反复尝试从任务队列中获取任务， getTask方法是阻塞方法， 当返回null时表示等待超时
            while (task != null || (task = getTask()) != null) {
                w.lock();
                // 通过中断关闭线程， 后文再做分析
                if ((runStateAtLeast(ctl.get(), STOP) ||
                     (Thread.interrupted() &&
                      runStateAtLeast(ctl.get(), STOP))) &&
                    !wt.isInterrupted())
                    wt.interrupt();
                try {
                    // ThreadPoolExecutor默认为空方法，留给子类实现功能
                    beforeExecute(wt, task);
                    Throwable thrown = null;
                    try {
                        // 调用真实的业务方法
                        task.run();
                    } catch (RuntimeException x) {
                        thrown = x; throw x;
                    } catch (Error x) {
                        thrown = x; throw x;
                    } catch (Throwable x) {
                        thrown = x; throw new Error(x);
                    } finally {
                        // ThreadPoolExecutor默认为空方法， 留给子类实现功能
                        afterExecute(task, thrown);
                    }
                } finally {
                    task = null;
                    w.completedTasks++;
                    w.unlock();
                }
            }
            completedAbruptly = false;
        } finally {
            // 工作线程结束， 将线程从workers中移除。 
            processWorkerExit(w, completedAbruptly);
        }
    }

}
```



### 关闭线程（池）

Runnable任务添加到线程池之后，子线程开始不断从任务队列获取新的任务来执行。当长时间有任务空闲，非核心线程就需要关闭。而当外部调用关闭线程池时，所有子线程都需要关闭。如何关闭? 下面来讨论

先来看工作线程超时关闭机制。线程运行过程中，`ThreadPoolExecutor.Worker#runWorker()`方法中有一个带超时时间的阻塞的`getTask()`方法，它负责不断从工作线程队列获取新的Runnable来执行，当getTask返回null时，线程结束。

```java
private Runnable getTask() {
    boolean timedOut = false; // Did the last poll() time out?

    for (;;) {
        // 线程池状态
        int c = ctl.get();
        int rs = runStateOf(c);

        // 下面这个条件判断比较绕， 可以将其转换一下:
        // ((rs >= SHUTDOWN && rs >= STOP) || 
        //  (rs >= SHUTDOWN && workQueue.isEmpty()))
        // 它的意思是:
        // 1.在线程池已SHUTDOWN(停止接受新任务)的前提下， 如果STOP(不再处理排队中的任务)返回null， 以令当前线程停止;
        // 2.在线程池已SHUTDOWN(停止接受新任务)的前提下， 如果工作队列为空， 当前 
        //   线程返回null， 线程停止(因为不会再有新的任务进来了)
        if (rs >= SHUTDOWN && (rs >= STOP || workQueue.isEmpty())) {
            decrementWorkerCount();
            return null;
        }

        // 工作线程数
        int wc = workerCountOf(c);

        // 获取新的Task超时，当前循环方法退出，线程即将关闭：
        // 1.当线程数超过核心线程数，直接退出
        // 2.当线程数未超核心线程数（即剩下的全是核心线程），根据配置来选择是否关闭
        boolean timed = allowCoreThreadTimeOut || wc > corePoolSize;
        if ((wc > maximumPoolSize || (timed && timedOut))
            && (wc > 1 || workQueue.isEmpty())) {
            // 满足上面条件时， 通过CAS减少工作线程数， 返回null关闭当前线程
            if (compareAndDecrementWorkerCount(c))
                return null;
            continue;
        }

        try {
            // 从任务队列中获取任务，这里的keepAliveTime就是创建线程池的时候配置的核心线程最大存活时间
            Runnable r = timed ?
                workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS) :
                workQueue.take();
            if (r != null)
                return r;
            timedOut = true;
        } catch (InterruptedException retry) {
            timedOut = false;
        }
    }
}
```

当然，上面只是单个线程的关闭，在线程池的生命周期中，会有许多线程独立关闭的过程。如果我们要关闭整个线程池呢？有两个方法`shutdown`和`shutdownNow`， 前者只是把`ctl`状态改为`SHUTDOWN`， 等待所有工作线程各自在下次`getTask`时发现关闭状态主动终止。 而后者则会主动中断`workers`中的所有线程。比较简单。 不再赘述。