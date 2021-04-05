---
title: "[简介]Protobuf编码原理"
date: 2019-06-15T04:28:20+08:00
featured_image: "http://minio.gogodjzhu.com/images/20210405_111637_9fca5a35-78bb-4ad5-b452-1680fc94bbb5.png"
images: ["http://minio.gogodjzhu.com/images/20210405_111637_9fca5a35-78bb-4ad5-b452-1680fc94bbb5.png"]
tags: ["中间件"]
draft: false
---

Protobuf是Google提出来的在网络间传输报文的协议，是对`json`，`xml`这些格式的替代方案。使用Protobuf的优势主要是:

- 缩小报文体积(json的1/10, xml的1/20)
- 报文减小带来传输效率的增加
- 编解码效率高(号称5~100倍的提高)

跟Json、xml是基于字符串的编码格式不同，protobuf是对字节流的编码。所以json、xml可以做到报文的自解析，比如看到一串这样的json:

```json
{
  "id":150
}
```

很显然我们可以其中只有字段: `id=150`

但是其带来的的代价是每个字符都占据一定空间，比如`{`，`}`这样的字符事实上并无实际含义，完全可以去除。另外一个方面，在json格式中，所有的内容均以字符类型保存。所以即便是简单的整形数字比如150也是保存为三个字符’1’、’5’、’0’。假设每个字符2字节(java char类型)，就是6个字节（下面我们可以看到protobuf只需要3个字节即可保存以上所有的信息）。

而Protobuf则提出了新的方案，直接对字节流进行编码，因为不管你是字符还是数字，不论你的长度类型为何，最终都是以字节流的方式进行编码保存。省掉了保存为字符串的空间浪费和类型转换，得到空间和性能的双重提升。



# 设计思想

那么Protobuf底层是如何实现这样一套协议的呢？首先让我们看看一个常见的使用protobuf进行通信的流程:

1. 定义.proto报文格式
2. 使用protoc工具解析.proto文件得到目标编程语言的编码器 和 解码器代码（事实上是同一份代码）。
3. 使用编码器编码目标内容得到其报文，即字节流。
4. 使用解码器解码字节流，这是跨语言的。比如用java编码得到的字节流报文，可以用于php/c等等语言实现的解码器解码。

例如下面是一份常见的proto报文定义:

```protobuf
message Test1 {
    optional int32 a = 1;
}
```

格式很简单，第一行定义了报文的名字，第二行则定义一个可选的(optional)字段a，他的类型为int32，以及很重要的字段序号1。

一般来说，一个报文相当于一个map，我们定义字段名称和类型相当于key，通过这个key来获取对应的value，即字段值。但是对于数据的保存来说，key-value是一个非常低效的设计。因为事实上相同报文的每一条消息，相同字段使用的都是相同的字段名，重复保存相同的字段名带来巨大的空间浪费。protobuf把报文设计成连续的字节流，字段名这些信息则保留在报文格式.proto文件中。而protobuf的报文实体保存的仅仅是有效的字段数据，非常节约存储空间。



# 实现细节

使用命令`protoc -IPATH=./ –java_out=./ test.proto`将前文定义的`Test1`proto报文格式，解析生成Java的编解码器后，我们这样使用Test1进行编码：

```java
public void encode() {
    Test.Test1.Builder simple = Test.Test1.newBuilder();
    simple.setA(150);//将字段a设为150
    FileOutputStream output = new FileOutputStream("output");//结果输出到文件
    simple.build().writeTo(output);
    output.close();
}
```

运行以上代码，我们得到一个名为output的文件，观察其十六进制/二进制的格式会是这样的:

```bash
// 08       96       01
// 00001000 10010110 00000001
```

怎么编码得到这串字节，就是protobuf协议的编码细节。

## 协议格式

前面我们介绍过，一般的协议都是按照map的方式进行结构，protobuf对其进行优化，把key和value都编码到一串字节流中，每个字段都以key+value的方式进行组织。类似于： key1:value1, key2:value2的方式。这还不够，因为完整的key名太长在报文中保存并无必要，protobuf把key1简化成一个字节，他的格式为:

```bash
key = 字段序号 << 3 | type
```

字段序号容易理解，就是报文中每个字段的序号，他是按1递增的。type，其实就是protobuf组织字节流的几种方式，知道type才知道如何在连续的字节流中进行切分（但是要知道每个字段具体的类型和值，还要结合protobuf解码器）。type有以下几种：

| Type | Meaning          | Used For                                                 |
| :--- | :--------------- | :------------------------------------------------------- |
| 0    | Varint           | int32, int64, uint32, uint64, sint32, sint64, bool, enum |
| 1    | 64-bit           | fixed64, sfixed64, double                                |
| 2    | Length-delimited | string, bytes, embedded messages, packed repeated fields |
| 3    | Start group      | groups (deprecated)                                      |
| 4    | End group        | groups (deprecated)                                      |
| 5    | 32-bit           | fixed32, sfixed32, float                                 |

因为test1报文中的a字段是int32类型，可以很简单知道它的“key”是这样得到的:

```bash
08(00001000) = 00000001(序号1) << 3 | 0
```

下面是Value，根据key我们可以知道此字段是一个Varint类型的字段，这个类型的字段都有一个统一的字节组织方式：每个字节的最高位表示是否有更多的字节，剩余的7bit的数据拼接起来即是实际的数据内容（按照小头字节序组织）。文字有点虚，我们来看a字段实际的报文：

```bash
// value => 10010110 00000001
//           0010110  0000001 // 最高位表示是否有跟多的字节
//         小头字节序,翻转字节顺序
//           0000001  0010110
//               拼接字节
//                 10010110
```

至此我们得到了a字段的值对应的二进制字节码为:10010110，解码器在解码的时候发现此字段的类型为int32，那么简单地将二进制转为10进制即可得到a的值为150。数字值越小，所需要的字节数往往越小，如果按照正常的编码方式，一个int32类型，需要32位即4个字节，这里只需要2个（其实是3个）。这也是protobuf能够实现数据高压缩比的另一个原因：

> protobuf针对每种类型的特点，对字节码进行了单独的设计，从而摆脱了每种类型有固定长度的束缚以节省空间。

更多的类型就不深入了，如果读者有兴趣可以自己进一步了解。



*参考文档：[https://developers.google.com/protocol-buffers/docs/encoding](https://developers.google.com/protocol-buffers/docs/encoding)*