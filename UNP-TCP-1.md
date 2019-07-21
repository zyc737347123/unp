# TCP/UDP/IP学习笔记

##TCP

###1.1 TCP的报文结构图示

TCP 是面向字节流的，但传送的数据单元却是报文段：
例如一个 100kb 的 HTML 文档需要传送到另外一台计算机，并不会整个文档直接传送过去，可能会切割成几个部分，比如四个分别为 25kb 的数据段。而每个数据段再加上一个 TCP 首部，就组成了 TCP 报文。一共四个 TCP 报文，发送到另外一个端。另外一端收到数据包，然后再剔除 TCP 首部，组装起来。等到四个数据包都收到了，就能还原出来一个完整的 HTML 文档了。

在 OSI 的七层协议中，第二层（数据链路层）的数据叫「Frame」，第三层（网络层）上的数据叫「Packet」，第四层（传输层）的数据叫「Segment」。TCP 报文 (Segment)，包括首部和数据部分。我们程序的数据首先会打到TCP的Segment中，然后TCP的Segment会打到IP的Packet中，然后再打到以太网Ethernet的Frame中，传到对端后，各个层解析自己的协议，然后把数据交给更高层的协议处理

下面来看看TCP报文头格式
![](https://coolshell.cn/wp-content/uploads/2014/05/TCP-Header-01.jpg)
![](https://s3.notfalse.net/wp-content/uploads/2017/03/12013524/TCP-Header-Format-SEQ.png)

###1.2 TCP报文详解

1. **源端口和目的端口 Port**

2. **序号 Sequence Number**

3. **确认号 Acknowledgemt Number**
4. **数据偏移 Offset**
5. **保留 Reserved**
6. **标志位 TCP Flags**
7. **窗口大小 Window Size**
8. **校验和 TCP Checksum**
9. **紧急指针 Urgent Pointer** 

