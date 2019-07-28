# TCP/UDP/IP学习笔记

##TCP报文

###1.1 TCP的报文结构图示

TCP 是面向字节流的，但传送的数据单元却是报文段：
例如一个 100kb 的 HTML 文档需要传送到另外一台计算机，并不会整个文档直接传送过去，可能会切割成几个部分，比如四个分别为 25kb 的数据段。而每个数据段再加上一个 TCP 首部，就组成了 TCP 报文。一共四个 TCP 报文，发送到另外一个端。另外一端收到数据包，然后再剔除 TCP 首部，组装起来。等到四个数据包都收到了，就能还原出来一个完整的 HTML 文档了。

在 OSI 的七层协议中，第二层（数据链路层）的数据叫「Frame」，第三层（网络层）上的数据叫「Packet」，第四层（传输层）的数据叫「Segment」。TCP 报文 (Segment)，包括首部和数据部分。我们程序的数据首先会打到TCP的Segment中，然后TCP的Segment会打到IP的Packet中，然后再打到以太网Ethernet的Frame中，传到对端后，各个层解析自己的协议，然后把数据交给更高层的协议处理

下面来看看TCP报文头格式
![](https://coolshell.cn/wp-content/uploads/2014/05/TCP-Header-01.jpg)
![](https://s3.notfalse.net/wp-content/uploads/2017/03/12013524/TCP-Header-Format-SEQ.png)

###1.2 TCP报文详解

1. **源端口和目的端口 Port**

   各占 2 个 字节，共 4 个字节。
   用来告知主机该报文段是来自哪里以及传送给哪个应用程序（应用程序绑定了端口）的。
   进行 TCP 通讯时，客户端通常使用系统自动选择的临时端口号，而服务器则使用知名服务端口号

2. **序号 Sequence Number**

   占 4 个字节。
   TCP 是面向字节流的，在一个 TCP 连接中传输的字节流中的每个字节都按照顺序编号。
   例如 100 kb 的 HTML 文档数据，一共 102400 (100 * 1024) 个字节，那么每一个字节就都有了编号，整个文档的编号的范围是 0 ~ 102399。

   序号字段值指的是**本报文段**所发送的数据的第一个字节的序号。
   那么 100kb 的 HTML 文档分割成四个等分之后，
   第一个 TCP 报文段包含的是第一个 25kb 的数据，0 ~ 25599 字节， 该报文的序号的值就是：0
   第二个 TCP 报文段包含的是第二个 25kb 的数据，25600 ~ 51199 字节，该报文的序号的值就是：25600
   ......

   根据 8 位 = 1 字节，那么 4 个字节可以表示的数值范围：[0, 2^32]，一共 2^32 (4294967296) 个序号。
   序号增加到最大值的时候，下一个序号又回到了 0.
   也就是说 TCP 协议可对 4GB 的数据进行编号，在一般情况下可保证当序号重复使用时，旧序号的数据早已经通过网络到达终点或者丢失了。

3. **确认号 Acknowledgemt Number**

   占 4 个字节。
   表示**期望收到对方下一个报文段的序号值**。
   TCP 的可靠性，是建立在「每一个数据报文都需要确认收到」的基础之上的。
   就是说，通讯的任何一方在收到对方的一个报文之后，都要发送一个相对应的「确认报文」，来表达确认收到。
   **那么，确认报文，就会包含确认号**。
   例如，通讯的一方收到了第一个 25kb 的报文，该报文的 序号值=0，那么就需要回复一个**确认报文**，其中的确认号 = 25600

4. **数据偏移 Offset**

   占 4 个bit。
   这个字段实际上是指出了 **TCP 报文段的首部长度** ，它指出了 TCP报文段的数据起始处 距离 TCP报文的起始处 有多远。（注意 数据起始处 和 报文起始处 的意思）

   一个数据偏移量 = 4 byte，由于 4 位二进制数能表示的最大十进制数字是 15，因此数据偏移的最大值是 60 byte，这也侧面限制了 TCP 首部的最大长度

5. **保留 Reserved**

   占6 个bit。
   保留为今后使用，但目前应置为 0

6. **标志位 TCP Flags**

   标志位，一共有 6 个，分别占 1 位，共 6 位 。
   每一位的值只有 0 和 1，分别表达不同意思

   - **紧急 URG (Urgent)**

     当 URG = 1 的时候，表示紧急指针（Urgent Pointer）有效。
     它告诉系统此报文段中有紧急数据，应尽快传送，而不要按原来的排队顺序来传送。
     URG 要与首部中的 紧急指针 字段配合使用

   - **确认 ACK (Acknowlegemt)**

     当 ACK = 1 的时候，确认号（Acknowledgemt Number）有效。
     一般称携带 ACK 标志的 TCP 报文段为「确认报文段」。
     **TCP 规定，在连接建立后所有传送的报文段都必须把 ACK 设置为 1**，如果一端在建立连接后没有发送过数据，ACK = ISN初始值 + 1

   - **推送 PSH (Push)**

     当 PSH = 1 的时候，表示该报文段高优先级，接收方 TCP 应该尽快推送给接收应用程序，而不用等到整个 TCP 缓存都填满了后再交付

   - **复位 RST (Reset)**

     当 RST = 1 的时候，表示 TCP 连接中出现严重错误，需要释放连接并重新建立连接。
     一般称携带 RST 标志的 TCP 报文段为「复位报文段」
     [产生RST的几种可能原因](https://my.oschina.net/costaxu/blog/127394)

   - **同步 SYN (SYNchronization)**

     当 SYN = 1 的时候，表明这是一个请求连接报文段。
     一般称携带 SYN 标志的 TCP 报文段为「同步报文段」。
     在 TCP 三次握手中的第一个报文就是同步报文段，在连接建立时用来同步序号。
     对方若同意建立连接，则应在响应的报文段的标志位中使 SYN = 1 和 ACK = 1

   - **终止 FIN (Finis)**

     当 FIN = 1 时，表示此报文段的发送方的数据已经发送完毕，并要求释放 TCP 连接。
     一般称携带 FIN 的报文段为「结束报文段」。
     在 TCP 四次挥手释放连接的时候，就会用到该标志

7. **窗口大小 Window Size**

   占 2 字节。
   该字段明确指出了现在允许对方发送的数据量，它告诉对方本端的 TCP 接收缓冲区还能容纳多少字节的数据，这样对方就可以控制发送数据的速度。
   窗口大小的值是指，从本报文段首部中的确认号算起，接收方目前允许对方发送的数据量。
   例如，假如确认号是 701 ，窗口字段是 1000。这就表明，从 701 号算起，发送此报文段的一方还有接收 1000 （字节序号是 701 ~ 1700） 个字节的数据的接收缓存空间

8. **校验和 TCP Checksum**

   占 2 个字节。
   由发送端填充，接收端对 TCP 报文段执行 CRC 算法，以检验 TCP 报文段在传输过程中是否损坏，如果损坏这丢弃。
   检验范围包括首部和数据两部分，这也是 TCP 可靠传输的一个重要保障

9. **紧急指针 Urgent Pointer** 

   占 2 个字节。
   仅在 URG = 1 时才有意义，它指出本报文段中的紧急数据的字节数。
   当 URG = 1 时，发送方 TCP 就把紧急数据插入到本报文段数据的**最前面**，而在紧急数据后面的数据仍是普通数据。
   因此，紧急指针指出了紧急数据的末尾在报文段中的位置
