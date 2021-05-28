---
title: "[译]Java垃圾回收器及对象引用"
date: 2020-02-05T12:29:01+08:00
description: "我们即将讨论Java中的垃圾回收概念和回收中会用到到几种对象引用类型"
tags: ["JAVA", "译文", "GC"]
categories: "Java原理系列"
draft: false
---

> 原文地址:[https://dzone.com/articles/java-garbage-collector-and-reference-objects](https://dzone.com/articles/java-garbage-collector-and-reference-objects)



在本文中，我们将讨论Java中的一些内存管理概念，重点是垃圾收集器与不同对象引用类型之间的交互。

这不是简单的入门文章，你应该预先了解`Java Heap`和`GC`的基础知识。 许多文章都很好地涵盖了该主题。我发现大多数文章都很好地介绍了Java内存，但是大都止步于对诸如对象引用类型的讨论。 我想尽我所能，但愿不要陷入同样的境地中。内存管理是高级开发人员面试问题的金矿。 “ Java管理自己的内存，我真的不必知道它是如何做到的。” 这或许是跟你身边朋友讨论时的说辞，但如果你跟一个苛刻的面试官也持有这样的态度，那只能祝你好运了…



# 类比

*使用类比的方式来讨论计算机概念往往非常简洁而且香。通常会充满“哦！呵呵！”的感叹。希望你在本文也有这样的体验。*

想象学校的自助餐厅。餐盘稀缺，但经理很聪明。他与他的员工一起制定了一系列的策略。目的是及时为所有饥饿的学生提供食物，而不会因盘子不足而使任何人吃不上饭。



## 策略0

**在学生吃完饭并离开自助餐厅时收集所有用过的盘子，洗涤后，提供给尚未吃饭的学生使用。**

因此，每当服务团队报告餐盘用完时，都会派出专门的服务员来收集所有用过的餐盘。只要正在使用盘子的学生离开自助餐厅，便会收集盘子。然后将这些盘子清洗后放在一起，以服务更多的学生。

该策略非常有效，经理对自己和他的员工感到满意。但很快，他意识到一些学生吃完饭了，却仍坚持与朋友聊天吹水。由于他的服务员只在学生离开桌子时才收集盘子，结果导致很多脏盘子都留在桌子上。仅仅因为吃完饭的学生仍然坐在那里，就导致自助餐厅经常遭遇“盘子危机”。



## 策略1

**经理只好另辟蹊径，他要求收盘子的服务员作出如下调整：只要学生吃完饭，就收集一个盘子，无论他们是否仍然坐在桌子上。**结果许多吃饭吃得慢的（话痨的）学生觉得这样的改变太影响就餐，投诉意见如雪片般飞向餐厅经理。该策略被证明为失败。



## 策略2

但作为一个聪明的人，经理马上又提出了一个更好想法：

- 如果学生是学生会或部门领导，就允许他们将餐盘保留任意长的时间，直到他们明确要求服务员收走餐盘或离开餐桌为止。
- 如果用餐的是学生会或部门领导的女友或男友，则让他们享有伴侣的特权。哪怕他们吃完饭后仍做在餐桌上不走，也不着急去收他们的盘子。除非餐厅可用盘子已达极限，而且没有别人（地位没有如此显贵的其他学生）的盘子可收时，才不得不回收他们使用过的盘子。
- 如果学生不是学生会或部门领导，并且跟这些人没有任何关系，就像大多数新生的情况一样，服务员应该对这些人的盘子随时保持关注。一旦他们吃完饭，不管他们如何要求，也不论他们是否仍坐在桌子上，立即回收他们的盘子。
- 最后，医生已经发给经理一份糖尿病学生的名单，要求了解这些学生每天最后一餐的确切时间，以便确定何时进行常规血糖水平检查。对这些学生，盘子的回收策略跟上一个类型的学生一样，但是在回收盘子的时候多做一步，记下回收盘子的时间（即他们吃完饭的时间）。

事实证明，这是一个极好的策略。

Java中的对象引用跟饭堂回收学生盘子一样，遵循类似的**分级的权限和强制措施**。下面我们开始深入讨论现实的技术细节。



# 强引用

在所有的Java程序中，对象实例都是保存和修改数据的依据：

```java
StringBuilder sb = new StringBuilder();
```

在上面这段代码中，`new`关键字创建了一个`StringBuilder`对象，并将对它的强引用存储在变量`sb`中。 **强引用**是我们创建的所有对象的默认强度级别，因此不会像即将讨论的其他引用类型那样使用任何特殊标签来标识它。

要像餐厅收盘子那样区分学生等级，我们需要`java.lang.ref`包中的封装类来封装新对象。

只要一个对象拥有`强引用`就肯定不符合垃圾回收的条件。 以我们的餐厅类比来看，这是`学生会或部门领导`对他们的盘子的强大占有欲。 用技术术语来说，该对象是`很容易被触达的`。因此只有将这些强引用全置为`null`，垃圾回收器才可以对这些对象进行回收，像下面这样：

```java
sb = null;
```



# 软引用

软引用通过`java.lang.ref.SoftReference`创建：

```java
StringBuilder sb = new StringBuilder();
SoftReference<StringBuilder> sbSoftRef = new SoftReference<>(sb);
sb = null;
```

在上面的代码段中，第一行创建了一个`StringBuilder`对象，该对象有`强引用`sb。 第二行在`sbSoftRef`中创建此对象的软引用。于是现在`StringBuilder`对象具有两个引用。

在此阶段，`StringBuilder`不符合垃圾回收条件。 但是，第三行使强引用无效，现在该对象仅具有软引用。

现在，该对象类似于学生会领导的男/女友桌子上放着的用过的盘子——只有在自助餐厅工作人员确定没有更多可用盘子的情况下，这个对象才会被回收。 从技术上讲，我们说该对象可以`轻柔地`到达。

在此阶段，我们仍然可以通过调用`SoftReference`对象的`get`方法来检索对该对象的强引用，如果该对象已经被回收，则该方法返回null：

```java
sb = sbSoftRef.get();
```



# 弱引用

弱引用通过`java.lang.ref.WeakReference`创建：

```java
StringBuilder sb = new StringBuilder();
WeakReference<StringBuilder> sbWeakRef = new WeakReference<>(sb);
sb = null;
```

在上面的代码段中，在第三行中取消了强引用之后，该对象立即可以被GC回收。

现在，该对象类似于没有特权的坐在一年级学生面前的旧盘子。 服务员可以立即收走它，而无需考虑学生是否仍坐在桌子旁。 从技术上讲，我们说该对象是`弱可及的`。

尽管我们仍然可以检索到该对象的强引用，但是机会窗口要比软引用小得多（因为很容易就被GC回收了）。 与其他情况相比，我们得到`null`的频率要高得多：

```java
sb = sbWeakRef.get（）;
```

**无论内存是否紧张，GC都会积极回收仅具有弱引用的对象。**



# 虚引用

虚引用可以通过`java.lang.ref.PhantomReference`创建：

```java
StringBuilder sb = new StringBuilder();
ReferenceQueue<StringBuilder> refQ = new ReferenceQueue<>();
PhantomReference<StringBuilder> sbPhantomRef = new PhantomReference<>(sb, refQ); 
sb = null;
```

在上面的代码段中，在第四行中取消了强引用之后，该对象立即可以使用GC。先忽略`ReferenceQueue`对象，稍后再讲。**首先请你记住，与软引用和弱引用可以独立使用不同，虚引用必须依赖`ReferenceQueue`而发挥作用。**

现在，该对象类似于患有糖尿病的一年级学生面前的旧盘子。可以被立即收走，而无需考虑学生是否仍然坐在桌子旁。但回收的时候会多做一件事：记录确切的收集时间并通知医生。

只要已注明时间的纸还没有被医生使用和丢弃，我们将其称为`幻影可到达的`对象。 （这很容易产生困惑，没关系，继续往下看）。

`PhantomReference`对象的get方法是无用的，因为它始终返回null。相对于软和弱引用，这进一步增强了虚引用的独特性。下一节将使这种独特性更加清晰。

虚引用旨在用作`Object.finalize()`方法的一种更灵活的替代方法。

## ReferenceQueue

顾名思义，ReferenceQueue是一个队列，保存的是几种类型的对象引用，即WeakReference，SoftReference和PhantomReference。

对象引用是否会加入队列取决于我们在创建对象引用时是否提供ReferenceQueue参数。除PhantomReference之外，并不强制要求提供这个参数的，甚至提供了也没有用。

```java
Object obj = new Object();
ReferenceQueue<Object> referenceQueue = new ReferenceQueue<Object>();

WeakReference<Object> weakReference = new WeakReference<Object>(obj, referenceQueue);
SoftReference<Object> softReference = new SoftReference<Object>(obj, referenceQueue);
PhantomReference<Object> phantomReference = new PhantomReference<Object>(obj, referenceQueue);
```

根据引用的类型，入队的时机会有所不同。但除了虚引用之外，我不打算再延伸讨论别的类型在这方面的内容。

垃圾回收器一旦完成了对虚引用所指对象的回收，该虚引用便被加入队列，此时已调用其`finalize()`方法。 `finalize()`方法是垃圾回收其在回收对象之前调用的一个对象方法——使得有机会在回收对象之前，释放对象在其生存期（在Java堆内的时期）中创建或使用的不受GC控制的资源。一个典型的例子是操作系统提供的文件句柄。为了演示，请看一下FileInputStream类的finalize方法：

```java
protected void finalize() throws IOException {
  if ((fd != null) &&  (fd != FileDescriptor.in)) {
    /* if fd is shared, the references in FileDescriptor
     * will ensure that finalizer is only called when
     * safe to do so. All references using the fd have
     * become unreachable. We can call close()
     */
    close();
  }
}
```

注意最后的调用的`close`方法，看起来很熟悉吗？这是Java老师告诉你需要放在try / catch / finally块的finally子句中的`fis.close()`。想知道这是否会导致重复调用？很好，请接着往下看！

注意到if判断，如果您已经在finally代码块中调用过`close()`，则fd将为null，这样`finalize()`就不会进入方法体导致重复调用。可以把`finalize()`方法理解成（尤其是在上面的`FileInputStream`这样的库/平台类）提供给开发人员的，忘记通过finally子句释放非堆资源时的保障。

那么，所有这些与`PhantomReference`和引用队列有什么关系?

事实上`finalize()`方法存在许多问题，这些问题几乎抵消了它试图提供的所有优点。实际上，Joshua Bloch在《Effective Java》一书中有详细的介绍，并且一些博客也广泛地讨论这个问题。不要以为他们只是在污蔑本来不错的API。除非您是即将上任的James Gosling（Java编程语言的共同创始人之一），否则我强烈建议您只听从他们的话，以善用资源。现在，继续阅读，差不多就结束了。

现在，是时候讨论为什么要引入虚引用和引用队列了。简单说，虚引用是“为`finalize()`提供更灵活的替代方案。”这样的尝试是否有效是一个很大的话题，但我敢打赌没有。

开发者在创建虚引用时如果提供了引用队列参数（在前述的例子代码中已经展示过了），那么在调用了对象的`finalize()`方法后，他的虚引用就会被加入引用队列。我们的业务代码需要负责遍历引用队列以跟踪此事件的发生。在业务代码中，我们可以手动释放资源，包括最终调用`phantomRef.clear()`方法以使强引用无效。



# 差异汇总

我知道本节的标题(译者注:原文为Differences Put Together)有点蹩脚，但请多多包涵。我将尝试汇总三种对象引用类型的差异：



## 入队列

在GC动作发生的时候（译者注：不同垃圾回收器会有不同的触发策略）软引用和弱引用会先被标记为「可回收」并放入队列，但是不会立即执行`finalize`方法进行实际的回收。而虚引用则在入队列的同时执行`finalize`方法。



## Reference.get()

软引用和弱引用的`get`方法都返回指向对象的强引用；如果该对象已被标记为可回收，则返回null。而虚引用的`get`方法始终返回null。这点区别导致当你拥有一个软引用或者弱引用时，只要对象未被回收（还在机会窗口内），你可以重新创建一个强引用。但是虚引用就做不到这一点。



## Reference.clear()

此方法在所有的三种引用类型中均可用。它将对应的强引用设置为null，这样可以在垃圾回收器扫描之前，显示地将对象标记为可回收。但这个方法对于软引用和弱引用并无意义，因为他们自身还占据着引用导致无法保证对象可以回收（除非达到各自的回收条件）。但在利用虚引用来主动释放资源时，就很有必要调用方法以回收强引用（因为当对象依赖的资源已经释放，对象也等于已经无效，强引用也就失效故需清空）。



## ReferenceQueue

对于软和弱引用，它是没有用的，但是没有它，虚引用却无法工作（原因前面已经讲过）。



# 用例

软引用：对内存敏感的缓存

弱引用：规范化映射

幻像引用：更灵活地替代`finalize()`



# 结论

在这篇文章中，尽管篇幅很长，但我试图讨论我关于GC与对象引用之间关系的一些研究。 我希望它至少能使你更好地理解引用及GC。

当然，你可能永远不必直接处理引用类型，除非你：

需要内存敏感的缓存？ （顺便说一下，这是一项非常复杂而细致的工作？）这时你应该考虑使用诸如`ehcache`专门用于缓存的库。

需要规范化的映射？ 只需使用`java.util.WeakHashMap`即可消除你的烦恼。

需要做对象回收前的清理工作吗？ 当然最好还是不要被虚引用提供的一些类似于`finalize()`的保证所欺骗，请坚持使用古老的try / catch / finally。