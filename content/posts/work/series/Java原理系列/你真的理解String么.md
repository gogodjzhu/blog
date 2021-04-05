---
title: "你真的理解Java String么?"
date: 2019-08-01T23:09:23+08:00
description: "java.lang.String大概是我们使用最多的一个类了，但对它的底层还是有很多东西值得我们去深究。这次就试图深入String，以后再不要问我String相关的问题。"
tags: ["JAVA"]
categories: "Java原理系列"
draft: false
---

## String类

`java.lang.String` 是`final`类，这意味着它内部的属性都被隐式地指定为`final`。对于`final`修饰的类，意味着它不可被继承。对于`final`修饰的属性，意味着不能修改.

> **final修饰的属性**
>
> 如果属性是基础类型，则不能改值。如果属性是对象引用，则不能改变所引用的地址.

在`String`内部共有2个重要属性:

```java
// 实际保存字符串的值
private final char value[];
// hash值
private int hash; // Default to 0
```

`char`数组用于保存字符串内部的每个字符，字符都是以`UTF-16`编码保存的。`char`被修饰为`private final`，内部所有操作均不暴露`char`，这是JVM刻意要将数组设计成不可变的对象。了解了这一点，面对创建了几个对象的问题，就可以手到擒来了.

```java
String a = "a"; // 创建了一个对象
String b = a + "b"; // 这里又创建一个新的对象，千万注意不是在原有对象基础上做修改
```

`hash`是字符串的hash值，通过对每个字符做数值计算得到。所以两个值相同的字符串的hash值肯定是一致的。



## String对象的保存位置

运行时通过`new String("value")`来创建，那么该对象会被保存在`堆内存`中。如果使用引号`"value"`来创建，则会将对象保存在`字符串常量池`(以下简称常量池)中。常量池的存在可以使得相同值的字符串在内存中仅保留一份拷贝，可以节省空间。**它的底层类似于一个线程安全的`HashMap`，相同Hash值的字符串落到一个bucket中退化为链表保存。所以保存在常量池中的字符串如果distinct数量庞大，就需要考虑扩大池的容量。**JVM提供了`-XX:StringTableSize=N`来进行配置。

> **题外话**
>
> String str = “value”; 这种操作涉及到自动装箱，在JVM装箱的过程中做了:
>
> 1. 检查常量池
> 2. 往常量池写入新对象
> 3. 返回常量池中的对象的引用.

```java
String str1 = new String("a");
String str2 = new String("a");
System.out.println(str1 == str2);
// false 两次调用String构造方法均生成了一个新的对象保存在堆中，不是同一个

String str3 = "a";
String str4 = "a";
System.out.println(str3 == str4);
// true 通过自动装箱创建的两个对象，均指向常量池中的相同地址

String str5 = "aa";
String str6 = str3 + "a";
System.out.println(str5 == str6);
// false str5通过自动装箱创建，保存在常量池中。
//       str6是在运行时计算得到新对象，保存在堆中
```

但是要注意，在不同的JDK版本中，常量池的实现是不同的

- JDK6 – 这个版本将常量池放在方法区中，由于方法区不能扩容，而且方法区gc也非常困难，很容易产生OOM
- JDK7 – 将常量池放在堆内存，这样做可以使得字符串可以跟普通对象一样维护。



#### String#intern方法

有了前面的铺垫，`intern()`方法的引入就非常自然了。我们通过`new String()`构造的对象都是保存在堆中的，所以重复的值会创建冗余的对象占用内存。这时就可以通过调用`intern()`方法来强制写入常量池，如果池内有同值字符串对象，则直接返回引用，否则将新对象保存在池中。



#### 高性能String Pool

结论说在前面，并不建议将业务构建在常量池之上。性能问题主要出在两个方面:

1. 当distinct字符串数量太多，退化成链表的hashmap性能极差
2. 并发情况下常量池的同步性能比较低(这个详情没有研究过，读者有兴趣可以深入研究原因)

所以如果确实需要使用字符串常量池，不妨通过`ConcurrentHashMap`这样的线程安全容器自己实现一套。