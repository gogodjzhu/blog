---
title: "[简介]Java字节码"
date: 2021-04-01T17:45:11+08:00
description: "简单介绍了如何从.java文件获取字节码，并简单介绍了虚方法在重写、重载以及泛型中的应用"
featured_image: "http://minio.gogodjzhu.com/images/20210403_120453_416044f6-3b96-4d81-a7ab-bbc056e39398.png"
images: ["http://minio.gogodjzhu.com/images/20210403_120453_416044f6-3b96-4d81-a7ab-bbc056e39398.png"]
tags: ["JAVA"]
categories: "Java原理系列"
draft: false
---

> 字节码是Java程序员走向进阶的必经之路。如果说一切业务问题在源码面前没有秘密，那么字节码就是带你走到秘密的背后，一窥Java底层的奥义。

## 如何获取字节码？

JVM装载类的方式有很多，最常见的做法就是利用`.class`文件，这是根据`.java`文件编译得到的。由于编译过程的存在，所以`.java`文件所表示的内容在`.class`可能发生变化。你可以通过`javap -v xxx.class`命令来获取指定`.class`可读的反编译格式，这就是字节码。

比如你经常用来跟世界打招呼的代码：

```java
public class HelloWorld {

    public static void main(String[] args) {
        System.out.println("hello world");
    }

}
```

它的字节码是这样的：

```java
Classfile /path/to/class/HelloWorld.class
  Last modified 2020-2-29; size 533 bytes
  MD5 checksum 2c60432c2c9027f3227d463e3827153e
  Compiled from "HelloWorld.java"
public class HelloWorld
  minor version: 0
  major version: 50
  flags: ACC_PUBLIC, ACC_SUPER
Constant pool:
   #1 = Methodref          #6.#20         // java/lang/Object."<init>":()V
   #2 = Fieldref           #21.#22        // java/lang/System.out:Ljava/io/PrintStream;
   #3 = String             #23            // hello world
   #4 = Methodref          #24.#25        // java/io/PrintStream.println:(Ljava/lang/String;)V
   #5 = Class              #26            // HelloWorld
   #6 = Class              #27            // java/lang/Object
   #7 = Utf8               <init>
   #8 = Utf8               ()V
   #9 = Utf8               Code
  #10 = Utf8               LineNumberTable
  #11 = Utf8               LocalVariableTable
  #12 = Utf8               this
  #13 = Utf8               LHelloWorld;
  #14 = Utf8               main
  #15 = Utf8               ([Ljava/lang/String;)V
  #16 = Utf8               args
  #17 = Utf8               [Ljava/lang/String;
  #18 = Utf8               SourceFile
  #19 = Utf8               HelloWorld.java
  #20 = NameAndType        #7:#8          // "<init>":()V
  #21 = Class              #28            // java/lang/System
  #22 = NameAndType        #29:#30        // out:Ljava/io/PrintStream;
  #23 = Utf8               hello world
  #24 = Class              #31            // java/io/PrintStream
  #25 = NameAndType        #32:#33        // println:(Ljava/lang/String;)V
  #26 = Utf8               HelloWorld
  #27 = Utf8               java/lang/Object
  #28 = Utf8               java/lang/System
  #29 = Utf8               out
  #30 = Utf8               Ljava/io/PrintStream;
  #31 = Utf8               java/io/PrintStream
  #32 = Utf8               println
  #33 = Utf8               (Ljava/lang/String;)V
{
  public HelloWorld();
    descriptor: ()V
    flags: ACC_PUBLIC
    Code:
      stack=1, locals=1, args_size=1
         0: aload_0
         1: invokespecial #1                  // Method java/lang/Object."<init>":()V
         4: return
      LineNumberTable:
        line 1: 0
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0       5     0  this   LHelloWorld;

  public static void main(java.lang.String[]);
    descriptor: ([Ljava/lang/String;)V
    flags: ACC_PUBLIC, ACC_STATIC
    Code:
      stack=2, locals=1, args_size=1
         0: getstatic     #2                  // Field java/lang/System.out:Ljava/io/PrintStream;
         3: ldc           #3                  // String hello world
         5: invokevirtual #4                  // Method java/io/PrintStream.println:(Ljava/lang/String;)V
         8: return
      LineNumberTable:
        line 4: 0
        line 5: 8
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0       9     0  args   [Ljava/lang/String;
}
SourceFile: "HelloWorld.java"
```



## 字节码的格式

从反编译得到字节码，你可以知道以下信息：

- 类的基础信息

  ![img](http://minio.gogodjzhu.com/images/20210403_101517_251ca0d2-bdfd-42b6-a2ea-340b9d9a3175.png)

- 常量池信息

  ![img](http://minio.gogodjzhu.com/images/20210403_101525_eafa726c-eb1b-4c45-a77e-a5ff7156c5d1.png)

- 类的方法列表

  ![img](http://minio.gogodjzhu.com/images/20210403_101531_f7b7697a-ec2a-4e30-b0be-8592536a394c.png)

第一次看到这样的字节码文件相信你会有点晕，有点怀疑自己简历里写的n年Java开发经验？莫方，一点一点来。先看看方法调用。



## 方法调用

关注到第65行，调用`System.out.println("Hello world");`方法反编译之后得到的字节码命令为`invokevirtual`，参数为`#4`。从常量池中可以找到`#4`对应的是`#24.#25`，然后又是一堆引用，最终指向的是打印日志的方法。

这就是JVM调用方法的过程。由于字节码是编译器通过java代码编译得到的，此时没有运行时环境，并不知道目标方法的具体内存地址，所以只能通过符号引用来表示该目标方法。符号引用包括了目标方法所在的类或者接口的名字，以及目标方法的方法名和方法描述符。启动JVM，调用具体方法时，会根据符号引用来获取目标方法以执行操作。

### 虚方法

由于字节码中没法包含运行时方法的具体地址，所以JVM设计了一套虚方法的机制来实现多态。具体来说，就是在方法定义中的调用采用虚方法调用（无法知道具体的方法，仅仅是虚拟的），运行时再通过调用者的实例类型来确定实际方法。

看一个例子：

```java
// MyInterface.java
public interface MyInterface {

    void interfaceInvoke();

}

// MyAbstractClass.java
public abstract class MyAbstractClass implements MyInterface {

    public void abstractInvokeImpl(){}

    public abstract void abstractInvoke();

}

// MyClass.java
public class MyClass extends MyAbstractClass{


    @Override
    public void interfaceInvoke() {

    }

    @Override
    public void abstractInvoke() {

    }

    public void invoke(){

    }

    public static void staticInvoke() {

    }

    private static void privateInvoke() {

    }

    public void main(MyClass myClass) {
        myClass.invoke();
        myClass.interfaceInvoke();
        myClass.abstractInvoke();
        myClass.abstractInvokeImpl();
    }
}
```

反编译后，关注`main`方法：

```java
  public void main(MyClass);
    descriptor: (LMyClass;)V
    flags: ACC_PUBLIC
    Code:
      stack=1, locals=2, args_size=2
         0: aload_1
         1: invokevirtual #2                  // Method invoke:()V
         4: aload_1
         5: invokevirtual #3                  // Method interfaceInvoke:()V
         8: aload_1
         9: invokevirtual #4                  // Method abstractInvoke:()V
        12: aload_1
        13: invokevirtual #5                  // Method abstractInvokeImpl:()V
        16: return
```

`aload_1`命令是个无参字节码指令，作用是从局部变量0中装载引用类型值入栈，后续的`invokevirtual`是通过虚方法的方式调用了`#2`符号引用指向的方法。跟踪每个`invokevirtual`方法的入参，都是同一个符号引用类型：`Methodref`。这类型的符号引用，JVM会按照如下步骤进行查找它的实际方法：

1. 在符号引用类（在这里即MyClass）中查找符合名字及描述符的方法
2. 在父类中寻找，直至Object类
3. 在实现的接口中寻找

另外值得注意的是，在main方法中我们使用了四种方式调用方法：

```java
myClass.invoke(); // 类的public方法
myClass.interfaceInvoke(); // 父接口定义的方法
myClass.abstractInvoke(); // 父类定义的抽象方法
myClass.abstractInvokeImpl(); // 父类定义的实现方法
```

而转化成字节码之后，其实都是一种类型的调用，即同为`invokeVirtual`虚方法调用。

> **其他方法调用**
>
> 除了虚方法调用，字节码还提供了其余的几种调用指令：
>
> invokestatic：用于调用静态方法。
> invokespecial：用于调用私有实例方法、构造器，以及使用 super 关键字调用父类的实例方法或构造器，和所实现接口的默认方法。
> invokevirtual：用于调用非私有实例方法。
> invokeinterface：用于调用接口方法。
> invokedynamic：用于调用动态方法。

## 虚方法与重写/重载

有了虚方法的知识积累，现在你可以对Java多态的重写和重载有新的认识。

对于重载，n个重载方法对应的是n个同名，但是不同参数列表的字节码方法。本质上他们是不同的几个方法。

而重写，是在子类中使用相同的方法名和参数列表定义的一个方法，它代替了父类的方法存在于子类。在调用时，由于虚方法调用是先从子类中寻找方法，所以调用的是子类的重写方法。

## 虚方法与泛型

Java的泛型是Java提供的一颗语法糖，在本质上它不同于C++是使用类型爆炸的方式创建多个实际的不同类型的类。java的做法是把所有的类都向上转型为Object进行操作，在必要的时候再进行转型。具体地：

```java
// Node.java
public class Node<T> {
    public T data;
    public Node(T data) { this.data = data; }
    public void setData(T data) {
        this.data = data;
    }
}

// MyNode.java
public class MyNode extends Node<Integer> {
    public MyNode(Integer data) { super(data); }
    public void setData(Integer data) {
        super.setData(data);
    }
}
```

Node所有使用泛型代替的地方，在字节码中都会被替换为基础的类型（在这里因为没有extends限定符所以基础类型为Object）：

```java
  public T data;
    descriptor: Ljava/lang/Object;
    flags: ACC_PUBLIC
    Signature: #8                           // TT;

  public com.demo.test.method.Node(T);
    descriptor: (Ljava/lang/Object;)V
    flags: ACC_PUBLIC
    Code:
      stack=2, locals=2, args_size=2
         0: aload_0
         1: invokespecial #1                  // Method java/lang/Object."<init>":()V
         4: aload_0
         5: aload_1
         6: putfield      #2                  // Field data:Ljava/lang/Object;
         9: return
```

而MyNode子类将泛型类型确定为Integer，因此在setData子类中必然存在一个方法名为`setData`，参数列表为`Integer`。但是由于它还实现了父类方法`setData(Object)`的存在，子类MyNode的方法`setData(Integer)`仅作为重载方法存在（不是重写）。那么下面这个方法就是可以编译通过的：

```java
public void main() {
    Node node = new MyNode(new Integer(2));
    node.setData(new Object());
}
```

只是逻辑上会存在矛盾：

1. 因为`node`实例的实际类型为`MyNode`，而因为泛型规定`setData`方法的入参类型为Integer，使用Object显然不满足。
2. 父类泛型使用的是基础类型Object，因此又必须保留父类的`setData(Object)`方法

为了解决这样的矛盾，JVM提出了桥接方法的概念。即泛型子类除了重载实现一个方法之外，还将重写父方法，并强制转型后调用子类的重载方法。

转化成java代码的逻辑就类似于：

```java
public void main() {
    Node node = new MyNode(new Integer(2));
    node.setData((Integer)new Object()); // 执行时会报转型失败异常
}
```

在字节码的层面就是：

```java
public void setData(java.lang.Integer); // 重载方法
    descriptor: (Ljava/lang/Integer;)V
    flags: ACC_PUBLIC
    Code:
      stack=2, locals=2, args_size=2
         0: aload_0
         1: aload_1
         2: invokespecial #2                  // Method com/demo/test/method/Node.setData:(Ljava/lang/Object;)V
         5: return

public void setData(java.lang.Object); // 重写父类方法
    descriptor: (Ljava/lang/Object;)V
    flags: ACC_PUBLIC, ACC_BRIDGE, ACC_SYNTHETIC // 在方法标志中也指出了这是一个 桥接方法(ACC_BRIDGE)
    Code:
      stack=2, locals=2, args_size=2
         0: aload_0
         1: aload_1
         2: checkcast     #3                  // class java/lang/Integer 类型转换
         5: invokevirtual #4                  // Method setData:(Ljava/lang/Integer;)V 调用重载方法
         8: return
```

也就是说，泛型的底层实现，其实就是利用重载和重写的组合。所谓的语法糖，即是没有创建新的底层功能（或者说命令），它仅仅利用已有的功能在字节码层面进行组合而来，创造一种“这是新功能”的假象。