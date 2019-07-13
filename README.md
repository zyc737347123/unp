# 网络知识学习笔记

##  1. Web协议概览
### 一个经典例子

用一个下单的过程，看看互联网世界的运行过程中，都使用了哪些网络协议

1. 首先输入**URL**，此时需要用到**DNS**将网址转换为服务器的**IP**

2. 浏览器使用**HTTP**（应用层）协议描述请求
    ![](https://static001.geekbang.org/resource/image/d8/c6/d8a65ca347ad26acc9f1de49b10320c6.png)

3. 经过应用层封装后，浏览器会将应用层的包交给下一层去完成，通过socket 编程来实现。下一层是传输层。传输层有两种协议，一种是无连接的协议**UDP**，一种是面向连接的协议**TCP**
    ![](https://static001.geekbang.org/resource/image/53/ee/53c753a7d49c9dfe3cfeb26497e47eee.png)

4. 传输层封装完毕后，浏览器会将包交给操作系统的网络层。网络层的协议是 **IP** 协议
    ![](https://static001.geekbang.org/resource/image/45/1b/459a421975b27f6187d2aa4673171f1b.png)

5. OS知道了服务器IP不在一个网段(通过子网掩码计算，目标IP是否在同一网段)，会将数据包发到网关（路由器），网关IP一般是OS启动的时候被**DHCP**服务器告知网关IP。本地通信需要通过MAC地址完成，OS为了将数据包发给网关，会使用**ARP**协议获得网关MAC地址。于是OS将IP包交给下层（MAC层/链路层），网卡再将包发出去。由于这个包里面是有 MAC 地址的，因而它能够到达网关
![](https://static001.geekbang.org/resource/image/cc/4f/cc02190ac57af7fb6c3839534f2b674f.png)

6. 网关（一般是路由器）收到包以后，根据路由表判断目标IP的转发路径。路由是边关，连接着两个国家（局域网），每个局域网内的设备都可以使用MAC地址通信
![](https://static001.geekbang.org/resource/image/f7/e2/f7ea602aec91c67b35e710fb72a975e2.png)
路由器往往是知道这些“路径”，因为路由器之间会经常沟通，确定两个IP间的最佳“路径”，这种的协议就是**路由协议**，常用的有**OSPF**和**BGP**
![](https://static001.geekbang.org/resource/image/b2/d4/b25ad7afba7b79331d95875dd0f451d4.png)

7. 最好到达目标机器后，目标机器的OS和应用会将数据包逐层拆开，在确认无误后，将上层数据包继续传递，可以用下图概括
![](https://static001.geekbang.org/resource/image/b4/3f/b465ccfafe333bfdfb9daf78f96e123f.png)