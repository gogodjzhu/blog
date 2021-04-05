---
title: "趣谈网络协议-学习笔记-02"
date: 2021-04-01T17:45:11+08:00
description: ""
tags: ["网络", "网课笔记"]
categories: "趣谈网络协议-学习笔记"
draft: true
---



TCP协议首部结构, 不计选项字段，长20byte(即160位)

![img](http://minio.gogodjzhu.com/images/20210403_104150_433379bb-3471-49ee-b0c7-d2cf88508015.png)

PSH标记位置为1表示希望尽快发送当前包。

TCP段包含的源端和目标端的端口号，用于寻找源发端和收端的应用程序。结合IP首部的源端/目标端的IP地址唯一确定一个TCP连接。

IP地址+端口号称为一个插口(Socket)

## 三次握手

握手的目标：

- 同步Sequence序列号
  初始序列号ISN(Initial Sequence Number)
- 交换TCP通讯参数
  如MSS，窗口比例因子，选择性确认，指定校验和算法

tcpdump -i lo port 3000 -c 3 -S // 抓头3个报文(-c 3)，以绝对序列号显示(-S)

![img](http://minio.gogodjzhu.com/images/20210403_104259_bcbd2e36-0b73-4e5c-85bf-f3fff35f46ce.png)

序列号是随机的，且两端不同。为了防止攻击，当然要同步完整的序列号本身也是一个难题。

### 三次握手的状态

![img](http://minio.gogodjzhu.com/images/20210403_104305_47c8deff-a8af-45b1-8974-3f20d66119f0.png)可以通过netstat查看

![img](http://minio.gogodjzhu.com/images/20210403_104310_bf8c6815-042c-42cc-a089-2fb0f8196925.png)

基于三次握手的优化：tcp_fastopen，可以将三次握手通过cookie优化轮数。

应用系统内核为一个服务端创建两个队列：SYN Queue和Accept Queue，前者负责接收TCP连接(完成三次握手)，后者把创建好的连接放在Accept Queue，等待应用程序通过accept()方法来获取。这两个队列是优先长度的，可以通过`/proc/sys/net/ipv4`目录下的对应参数来做配置。

存在一种伪造`SYN`报文的SYN攻击，会打爆Accept Queue。可以通过tcp_syncookies(默认为1开启)来进行优化，当SYN队列满的时候启用cookie。这样可以规避只有SYN报文的攻击。

![img](http://minio.gogodjzhu.com/images/20210403_104316_a90469a6-f835-4c0e-a82f-eac7e36ec131.png)

三次握手的数据仅包含TCP头部

![img](http://minio.gogodjzhu.com/images/20210403_104322_e26a8346-03a7-49e8-b041-8ba5be9653ee.png)

SYN报文段作为命令报文，在报文的`数据段`实际无内容，但当作长度为1处理，发送ACK的时候需要做`+1`处理。

正常发送的过程中，响应端的ack字段取上一次接收到报文的`sequence number + n`，其中n为上一次收到报文的`数据段`字节数。

在SYN数据段中通常会加上一个MSS(Max Segment Size)可选项表示TCP传输的字节数大小，也就是payload的大小。一般为`MTU(一般为1500byte)-(IP头部20byte+TCP头部20byte)=2460byte`。

## 数据确认与滑动窗口

设计的目的是：

- 跟踪字节流中的每个字节的顺序
- 保证接收端能够在有效接收字节的前提下还不被打爆

SYC/ACK序号是基于字节的序号，而不是报文序号。在发送发出一个报文后，接收方确认接收后会对该报文的SYN序号加1作为ACK进行确认。多个连续的报文接收之后接收方可以统一确认（发送最大的序号）。

发送在发了一个包之后，启动定时器进行超时判断，并进行重发。

由于SYN/ACK都只有32位长度，换成字节数即4G，超过最大值的包会进行序号复用。这就带来另外一个问题：序列号回绕。回绕之后的ACK会跟重发的包的ACK相同，这个时候就利用TCP中的Timestamps Option来进行交叉验证。

同时Timestamps还用来计算RTT(Round-Trip Time)值。后者是计算RTO(Retransmission TimeOut)的基础。算法略。

![img](http://minio.gogodjzhu.com/images/20210403_104327_a193bcb3-6838-4e6c-90c2-c8257ceb01c1.png)

滑动窗口的初始大小是根据MTU来计算的。

## 四次挥手

![img](http://minio.gogodjzhu.com/images/20210403_104332_bff9a7c4-b799-46c9-b41a-a657e2ce5879.png)

常见问题：为什么服务器不将对客户端FIN的ACK与自己的FIN合并，从而将报文段数从4减少为3？
答: 作为被动接收FIN结束的服务器，在收到FIN的时候可能还有数据需要发送给客户端，所以不能马上发送FIN。
但对客户端FIN的应答也不能等，否则客户端会判断超时重发FIN

一个完整的tcp通信流程抓包如下:

![img](http://minio.gogodjzhu.com/images/20210403_104339_7aad6f4c-6ba8-4813-af01-009138ae1a3d.png)

## 2MSL(2 **Maximum Segment Lifetime**)

主动断开的一侧为A，被动断开的一侧为B。

**第一个消息：A发FIN
第二个消息：B回复ACK
第三个消息：B发出FIN**

此时此刻：B单方面认为自己与A达成了共识，即双方都同意关闭连接。

此时，B能释放这个TCP连接占用的内存资源吗？**不能，B一定要确保A收到自己的ACK、FIN。**

所以B需要静静地等待A的第四个消息的到来：

**第四个消息：A发出ACK，用于确认收到B的FIN**

当B接收到此消息，即认为双方达成了同步：双方都知道连接可以释放了，此时B可以安全地释放此TCP连接所占用的内存资源、端口号。

所以**被动关闭的B无需任何wait time，直接释放资源。**

但，A并不知道B是否接到自己的ACK，A是这么想的：

1）如果B没有收到自己的ACK，会超时重传FiN

那么A再次接到重传的FIN，会再次发送ACK

2）如果B收到自己的ACK，也不会再发任何消息，包括ACK

无论是1还是2，A都需要等待，要取这两种情况等待时间的最大值，**以应对最坏的情况发生**，这个最坏情况是：

去向ACK消息最大存活时间（MSL) + 来向FIN消息的最大存活时间(MSL)。

这恰恰就是**2MSL( Maximum Segment Life)。**

等待2MSL时间，A就可以放心地释放TCP占用的资源、端口号，**此时可以使用该端口号连接任何服务器。为何一定要等2MSL？**
**如果不等，释放的端口可能会重连刚断开的服务器端口，这样依然存活在网络里的老的TCP报文可能与新TCP连接报文冲突，造成数据冲突，为避免此种情况，需要耐心等待网络老的TCP连接的活跃报文全部死翘翘，2MSL时间可以满足这个需求（尽管非常保守）！**

## TCP常用配置

Linux系统参数