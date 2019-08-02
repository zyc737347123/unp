# TCP/UDP/IP学习笔记

##TCP可靠性

TCP 是一种提供可靠性交付的协议。也就是说，通过 TCP 连接传输的数据，无差错、不丢失、不重复、并且按序到达。TCP的可靠性通过下面几种技术实现

### 2.1 TCP重传

TCP要保证所有的数据包都可以到达，所以，必需要有重传机制。

注意，接收端给发送端的Ack确认只会确认最后一个连续的包，比如，发送端发了1,2,3,4,5一共五份数据，接收端收到了1，2，于是回ack 3，然后收到了4（注意此时3没收到），此时的TCP会怎么办？我们要知道，因为正如前面所说的，**SeqNum和Ack是以字节数为单位，所以ack的时候，不能跳着确认，只能确认最大的连续收到的包**，不然，发送端就以为之前的都收到了

#### 超时重传

一种接收方是不回ack，死等3，当发送方发现收不到3的ack超时后，会重传3。一旦接收方收到3后，会ack 回 4——意味着3和4都收到了。

但是，这种方式会有比较严重的问题，那就是因为要死等3，所以会导致4和5即便已经收到了，而发送方也完全不知道发生了什么事，因为没有收到Ack，所以，发送方可能会悲观地认为也丢了，所以有可能也会导致4和5的重传。

对此发送方有两种选择：

- 一种是仅重传timeout的包。也就是第3份数据。
- 另一种是重传timeout后所有的数据，也就是第3，4，5这三份数据。

这两种方式有好也有不好。第一种会节省带宽，但是慢，第二种会快一点，但是会浪费带宽，也可能会有无用功。但总体来说都不好。因为都在等timeout，timeout可能会很长

#### TCP的RTT

我们知道Timeout的设置对于重传非常重要。

- 设长了，重发就慢，丢了老半天才重发，没有效率，性能差；
- 设短了，会导致可能并没有丢就重发。于是重发的就快，会增加网络拥塞，导致更多的超时，更多的超时导致更多的重发。

而且，这个超时时间在不同的网络的情况下，根本没有办法设置一个固定值。只能动态地设置。 为了动态地设置，TCP引入了RTT——**Round Trip Time**，也就是一个数据包从发出去到回来的时间。这样发送端就大约知道需要多少的时间，从而可以方便地设置Timeout——RTO（**Retransmission TimeOut**），以让我们的重传机制更高效。 听起来似乎很简单，好像就是在发送端发包时记下t0，然后接收端再把这个ack回来时再记一个t1，于是RTT = t1 – t0。没那么简单，这只是一个采样，不能代表普遍情况

[几种RTT算法](https://coolshell.cn/articles/11609.html)

#### 快速重传

因为超时重传可能耗时很久，于是，TCP引入了一种叫**Fast Retransmit** 的算法，**不以时间驱动，而以数据驱动重传**。也就是说，如果，包没有连续到达，就ack最后那个可能被丢了的包，如果发送方连续收到3次相同的ack，就重传。Fast Retransmit的好处是不用等timeout了再重传。

比如：如果发送方发出了1，2，3，4，5份数据，第一份先到送了，于是就ack回2，结果2因为某些原因没收到，3到达了，于是还是ack回2，后面的4和5都到了，但是还是ack回2，因为2还是没有收到，于是发送端收到了三个ack=2的确认，知道了2还没有到，于是就马上重转2。然后，接收端收到了2，此时因为3，4，5都收到了，于是ack回6。示意图如下：
![](https://coolshell.cn/wp-content/uploads/2014/05/FASTIncast021.png)

Fast Retransmit只解决了一个问题，就是timeout的问题，它依然面临一个艰难的选择，就是，是重传之前的一个还是重传所有的问题。对于上面的示例来说，是重传#2呢还是重传#2，#3，#4，#5呢？因为发送端并不清楚这连续的3个ack(2)是谁传回来的？也许发送端发了20份数据，是#6，#10，#20传来的呢。这样，发送端很有可能要重传从2到20的这堆数据（这就是某些TCP的实际的实现）。可见，这是一把双刃剑

#### SACK方法

另外一种更好的方式叫：**Selective Acknowledgment (SACK)**（参看[RFC 2018](http://tools.ietf.org/html/rfc2018)），这种方式需要在TCP头里加一个SACK的东西，ACK还是Fast Retransmit的ACK，SACK则是汇报收到的数据碎版。参看下图：
![](https://coolshell.cn/wp-content/uploads/2014/05/tcp_sack_example-1024x577.jpg)

这样，在发送端就可以根据回传的SACK来知道哪些数据到了，哪些没有到。于是就优化了Fast Retransmit的算法。当然，这个协议需要两边都支持。在 Linux下，可以通过**tcp_sack**参数打开这个功能（Linux 2.4后默认打开）。

这里还需要注意一个问题——**接收方Reneging，所谓Reneging的意思就是接收方有权把已经报给发送端SACK里的数据给丢了**。这样干是不被鼓励的，因为这个事会把问题复杂化了，但是，可能会有些极端情况导致接受方这样做，比如要把内存给别的更重要的东西。**所以，发送方也不能完全依赖SACK，还是要依赖ACK，并维护Time-Out，如果后续的ACK没有增长，那么还是要把SACK的东西重传，另外，接收端这边永远不能把SACK的包标记为Ack。**

注意：SACK会消费发送方的资源，试想，如果一个攻击者给数据发送方发一堆SACK的选项，这会导致发送方开始要重传甚至遍历已经发出的数据，这会消耗很多发送端的资源。详细的东西请参看《[TCP SACK的性能权衡](http://www.ibm.com/developerworks/cn/linux/l-tcp-sack/)》

#### Duplicate SACK – 重复收到数据的问题

Linux2.4后默认打开选项，[DSACK详解](https://www.twblogs.net/a/5c168224bd9eee5e40bbb790)

引入D-SACK的目的是使TCP進行更好的流控，具體來說有以下幾個好處：

1. 讓發送方知道，是發送的包丟了，還是返回的ACK包丟了；

2. 網絡上是否出現了包失序；

3. 數據包是否被網絡上的路由器複製並轉發了

4. 是不是自己的timeout太小了，導致重傳

通過D-SACK這種方法，發送方可以更仔細判斷出當前網絡的傳輸情況，可以發現數據段被網絡複製、錯誤重傳、ACK丟失引起的重傳、重傳超時等異常的網絡狀。

**D-SACK使用了SACK的第一個段來做標誌**，**如何判斷D-SACK**：

1. 如果SACK的第一個段的範圍被ACK所覆蓋，那麼就是D-SACK

2. 如果SACK的第一個段的範圍被SACK的第二個段覆蓋，那麼就是D-SACK

D-SACK的規則如下：

1. **第一個block將包含重複收到的報文段的序號**
2. 跟在D-SACK之後的SACK將按照SACK的方式工作
3. **如果有多個被重複接收的報文段，則D-SACK只包含其中第一個**

### 2.2 滑动窗口

我们都知道，**TCP必需要解决的可靠传输以及包乱序（reordering）的问题**，所以，**TCP必需要知道网络实际的数据处理带宽或是数据处理速度，这样才不会引起网络拥塞，导致丢包，所以TCP需要做网络流量控制**。TCP引入了一些技术和设计来做网络流控，Sliding Window(滑动窗口)是其中一个技术。前面我们说过，**TCP头里有一个字段叫Window，又叫Advertised-Window(通告窗口)，这个字段是接收端告诉发送端自己还有多少缓冲区可以接收数据**。**于是发送端就可以根据这个接收端的处理能力来发送数据，而不会导致接收端处理不过来**

下面我们来看一下发送方的滑动窗口示意图：
![](https://coolshell.cn/wp-content/uploads/2014/05/tcpswwindows.png)

上图中分成了四个部分，分别是：（其中那个黑模型就是滑动窗口）

- \#1已收到ack确认的数据。

- \#2发还没收到ack的。

- \#3在窗口中还没有发出的（接收方还有空间）。

- \#4窗口以外的数据（接收方没空间）

下面是个滑动后的示意图（收到36的ack，并发出了46-51的字节）：
![](https://coolshell.cn/wp-content/uploads/2014/05/tcpswslide.png)

下面我们来看一个接受端控制发送端的图示：
![](https://coolshell.cn/wp-content/uploads/2014/05/tcpswflow.png)

#### Zero Window

上图，我们可以看到一个处理缓慢的Server（接收端）是怎么把Client（发送端）的TCP Sliding Window给降成0的。此时，你一定会问，如果Window变成0了，TCP会怎么样？是不是发送端就不发数据了？是的，发送端就不发数据了，你可以想像成“Window Closed”，那你一定还会问，如果发送端不发数据了，接收方一会儿Window size 可用了，怎么通知发送端呢？

解决这个问题，TCP使用了Zero Window Probe技术，缩写为ZWP，也就是说，发送端在窗口变成0后，会发ZWP的包给接收方，让接收方来ack他的Window尺寸，一般这个值会设置成3次，每次大约30-60秒（不同的实现可能会不一样）。如果3次过后还是0的话，**有的TCP实现就会发RST把链接断了**。

**注意**：**只要有等待的地方都可能出现DDoS攻击**，Zero Window也不例外，一些攻击者会在和HTTP服务端建好连接发完GET请求后，就把Window设置为0，然后服务端就只能等待进行ZWP，于是攻击者会并发大量的这样的请求，把服务器端的资源耗尽。（关于这方面的攻击，大家可以移步看一下[Wikipedia的SockStress词条](http://en.wikipedia.org/wiki/Sockstress)）

另外，Wireshark中，你可以使用tcp.analysis.zero_window来过滤包，然后使用右键菜单里的follow TCP stream，你可以看到ZeroWindowProbe及ZeroWindowProbeAck的包。

#### Silly Window Syndrome

[Silly Window Syndrome](https://wiki.kfd.me/wiki/%E7%B3%8A%E6%B6%82%E7%AA%97%E5%8F%A3%E7%BB%BC%E5%90%88%E7%97%87)翻译成中文就是“糊涂窗口综合症”。正如你上面看到的一样，如果我们的接收方太忙了，来不及取走Receive Windows里的数据，那么，就会导致发送方越来越小。到最后，如果接收方腾出几个字节并告诉发送方现在有几个字节的window，而我们的发送方会义无反顾地发送这几个字节。

要知道，我们的TCP+IP头有40个字节，为了几个字节，要达上这么大的开销，这太不经济了。

另外，你需要知道网络上有个MTU，对于以太网来说，MTU是1500字节，除去TCP+IP头的40个字节，真正的数据传输可以有1460，这就是所谓的MSS（Max Segment Size）注意，TCP的RFC定义这个MSS的默认值是536，这是因为 [RFC 791](http://tools.ietf.org/html/rfc791)里说了任何一个IP设备都得最少接收576尺寸的大小（实际上来说576是拨号网络的MTU，而576减去IP头的20个字节就是536）。

**如果你的网络包可以塞满MTU，那么你可以用满整个带宽，如果不能，那么你就会浪费带宽**。（大于MTU的包有两种结局，一种是直接被丢了，另一种是会被重新分块打包发送） 你可以想像成一个MTU就相当于一个飞机的最多可以装的人，如果这飞机里满载的话，带宽最高，如果一个飞机只运一个人的话，无疑成本增加了，也而相当二。

所以，**Silly Windows Syndrome这个现像就像是你本来可以坐200人的飞机里只做了一两个人**。 要解决这个问题也不难，就是避免对小的window size做出响应，直到有足够大的window size再响应，这个思路可以同时实现在sender和receiver两端。

- 如果这个问题是由Receiver端引起的，那么就会使用 David D Clark’s 方案。在receiver端，如果收到的数据导致window size小于某个值，可以直接ack(0)回sender，这样就把window给关闭了，也阻止了sender再发数据过来，等到receiver端处理了一些数据后windows size 大于等于了MSS，或者，receiver buffer有一半为空，就可以把window打开让send 发送数据过来（会不会导致Zero Window，进而导致服务端主动关闭连接？）

- 如果这个问题是由Sender端引起的，那么就会使用著名的 [Nagle’s algorithm](http://en.wikipedia.org/wiki/Nagle's_algorithm)。这个算法的思路也是**延时处理**，算法伪代码如下：

```
  if there is new data to send
    if the window size >= MSS and available data is >= MSS
      send complete MSS segment now
    else
      if there is unconfirmed data still in the pipe
        enqueue data in the buffer until an acknowledge is received
      else
        send data immediately
      end if
    end if
  end if
```

另外，Nagle算法默认是打开的，所以，对于一些需要小包场景的程序——**比如像telnet或ssh这样的交互性比较强的程序，你需要关闭这个算法**。你可以在Socket设置TCP_NODELAY选项来关闭这个算法（关闭Nagle算法没有全局参数，需要根据每个应用自己的特点来关闭）

```c
setsockopt(sock_fd, IPPROTO_TCP, TCP_NODELAY, (char *)&value,sizeof(int));
```

另外，网上有些文章说TCP_CORK的socket option也是关闭Nagle算法，这不对。**TCP_CORK其实是更激进的提供网络利用率的算法，完全禁止小包发送，而Nagle算法没有禁止小包发送，只是禁止了大量的小包发送（只要小包得到ACK，Nagle就可以继续发送小包）**。最好不要两个选项都设置。[更多关于糊涂窗口综合症的解释](https://www.cnblogs.com/zhaoyl/archive/2012/09/20/2695799.html)

### 2.3 拥塞处理

上面我们知道了，TCP通过Sliding Window来做流控（Flow Control），但是TCP觉得这还不够，因为**Sliding Window只依赖于连接的发送端和接收端，其并不知道网络中间发生了什么**。TCP的设计者觉得，一个伟大而牛逼的协议仅仅做到流控并不够，因为流控只是网络模型4层以上的事，TCP的还应该更聪明地知道整个网络上的事。

具体一点，我们知道TCP通过一个timer采样了RTT并计算RTO，但是，**如果网络上的延时突然增加，那么，TCP对这个事做出的应对只有重传数据，但是，重传会导致网络的负担更重，于是会导致更大的延迟以及更多的丢包，于是，这个情况就会进入恶性循环被不断地放大。试想一下，如果一个网络内有成千上万的TCP连接都这么行事，那么马上就会形成“网络风暴”，TCP这个协议就会拖垮整个网络，**这是一个灾难。

所以，TCP不能忽略网络上发生的事情，而无脑地一个劲地重发数据，对网络造成更大的伤害。对此TCP的设计理念是：**TCP不是一个自私的协议，当拥塞发生的时候，要做自我牺牲。就像交通阻塞一样，每个车都应该把路让出来，而不要再去抢路了。**

关于拥塞控制的论文请参看《[Congestion Avoidance and Control](http://ee.lbl.gov/papers/congavoid.pdf)》(PDF)

**拥塞控制**主要是四个算法：**1）慢启动**，**2）拥塞避免**，**3）拥塞发生**，**4）快速恢复**。这四个算法不是一天都搞出来的，这个四算法的发展经历了很多时间，到今天都还在优化中

- 1988年，TCP-Tahoe 提出了1）慢启动，2）拥塞避免，3）拥塞发生时的快速重传
- 1990年，TCP Reno 在Tahoe的基础上增加了4）快速恢复

#### 慢热启动算法 – Slow Start
首先，我们来看一下TCP的慢热启动。慢启动的意思是，刚刚加入网络的连接，一点一点地提速，不要一上来就像那些特权车一样霸道地把路占满。新同学上高速还是要慢一点，不要把高速上已有的秩序给搞乱了。

慢启动的算法如下(**cwnd**全称**Congestion Window**)：

1. 连接建好的开始先初始化cwnd = 1，表明可以传一个MSS(Max Segment Size)大小的数据。

2. 每当收到一个ACK，cwnd++; 呈线性上升

3. 每当过了一个RTT，cwnd = cwnd*2; 呈指数让升

4. 还有一个ssthresh（**slow start threshold**），是一个上限，当cwnd >= ssthresh时，就会进入“拥塞避免算法”

所以，我们可以看到，如果网速很快的话，ACK也会返回得快，RTT也会短，那么，这个慢启动就一点也不慢。下图说明了这个过程

![](https://coolshell.cn/wp-content/uploads/2014/05/tcp.slow_.start_.jpg)

这里需要提一下的是一篇Google的论文《[An Argument for Increasing TCP’s Initial Congestion Window](http://static.googleusercontent.com/media/research.google.com/zh-CN//pubs/archive/36640.pdf)》Linux 3.0后采用了这篇论文的建议——把cwnd 初始化成了 10个MSS。而Linux 3.0以前，比如2.6，Linux采用了[RFC3390](http://www.rfc-editor.org/rfc/rfc3390.txt)，cwnd是跟MSS的值来变的，如果MSS< 1095，则cwnd = 4；如果MSS>2190，则cwnd=2；其它情况下，则是3。

#### 拥塞避免算法 – Congestion Avoidance

前面说过，还有一个ssthresh（**slow start threshold**），是一个上限，当cwnd >= ssthresh时，就会进入“拥塞避免算法”。**一般来说ssthresh的值是65535，单位是字节**，当cwnd达到这个值时后，算法如下：

1. 收到一个ACK时，cwnd = cwnd + 1/cwnd

2. 当每过一个RTT时，cwnd = cwnd + 1

这样就可以避免增长过快导致网络拥塞，慢慢的增加调整到网络的最佳值。很明显，是一个线性上升的算法。

#### 拥塞状态时的算法(拥塞发生)

什么情况是拥塞发生，TCP认为出现丢包,需要重传就代表当前网络拥塞，TCP通过**duplicate acknowledgement**和**timeout**判断是否丢包，下面一段文字概述了拥塞发生时TCP的处理方式
[Congestion is detected either by receipt of a duplicate acknowledgement or timeout signal. Once this happens, the TCP sender decreases the send rate by decreasing the congestion window size by a factor determined by the algorithm used. The maximum amount of unacknowledged data that the source can send is the lower of the two windows.](https://blog.stackpath.com/glossary-cwnd-and-rwnd/)

总的来说，当丢包的时候，会有两种情况：

1. 等到RTO超时，重传数据包。TCP认为这种情况太糟糕，反应也很强烈。

  - sshthresh =  cwnd /2
  - cwnd 重置为 1
  - 进入慢启动过程

2. Fast Retransmit算法，也就是在收到3个duplicate ACK时就开启重传，而不用等到RTO超时。

  - TCP Tahoe的实现和RTO超时一样。

  - TCP Reno的实现是：
    - cwnd = cwnd /2
    - sshthresh = cwnd
    - 进入快速恢复算法——Fast Recovery

上面我们可以看到RTO超时后，sshthresh会变成cwnd的一半，这意味着，如果cwnd<=sshthresh时出现丢包，那么TCP的sshthresh就会减了一半，然后cwnd又很快地以指数级增涨爬到这个地方时，就会成慢慢的线性增涨。我们可以看到，TCP是怎么通过这种强烈地震荡快速而小心地找到网络流量的平衡点

#### 快速恢复算法 – Fast Recovery

这个算法定义在[RFC5681](http://tools.ietf.org/html/rfc5681)。快速重传和快速恢复算法一般同时使用。快速恢复算法是认为，你还有3个Duplicated Acks说明网络也不那么糟糕，所以没有必要像RTO超时那么强烈。 **也就是说，快速恢复是为了完成快速重传后，进入了拥塞避免阶段而不是慢启动阶段**。注意，正如前面所说，进入Fast Recovery之前，cwnd 和 sshthresh已被更新：

- cwnd = cwnd /2
- sshthresh = cwnd

然后，真正的Fast Recovery算法如下：

- cwnd = sshthresh  + 3 * MSS （3的意思是确认有3个数据包被收到了）
- 重传Duplicated ACKs指定的数据包
- 如果再收到 duplicated Acks，那么cwnd = cwnd +1
- 如果收到了新的Ack，那么，cwnd = sshthresh ，然后就进入了拥塞避免的算法了

### 2.4 拥堵窗口和滑动窗口

滑动窗口（sliding window）是接收方的流量控制。
拥塞控制（congestion control）是发送方的流量控制

滑动窗口和拥塞窗口是在解决两个正交的问题，只不过方法上都是在调整主机发送数据包的速率。

滑动窗口是解决Flow Control的问题，就是如果接收端和发送端对数据包的处理速度不同，如何让双方达成一致。

而拥塞窗口是解决多主机之间共享网络时出现的拥塞问题的一个修正。客观来说网络信道带宽不可能允许所有主机同时全速通信，所以如果全部主机都全速发送数据包，导致网络流量超过可用带宽，那么由于TCP的设计数据包会大量丢失，于是由于重传机制的触发会进一步加剧拥塞，潜在的导致网络不可用

[What is CWND and RWND?](https://blog.stackpath.com/glossary-cwnd-and-rwnd/)
[既然有了滑动窗口，为什么还要有等同于滑动窗口的拥塞窗口？](https://www.zhihu.com/question/264518499)
[流量控制与拥塞控制的区别。为什么要把流量控制与拥塞控制分为两个名词？](https://www.zhihu.com/question/38749788)

## 参考文献

- [TCP 的那些事儿-上](https://coolshell.cn/articles/11564.html)
- [TCP 的那些事儿-下](https://coolshell.cn/articles/11609.html)
- [What is CWND and RWND?](https://blog.stackpath.com/glossary-cwnd-and-rwnd/)
- [TCP快速重传与快速恢复原理分析](https://blog.csdn.net/zhangskd/article/details/7174682)
- [TCP重点系列之拥塞状态机](https://allen-kevin.github.io/2017/04/19/TCP%E9%87%8D%E7%82%B9%E7%B3%BB%E5%88%97%E4%B9%8B%E6%8B%A5%E5%A1%9E%E7%8A%B6%E6%80%81%E6%9C%BA/)

