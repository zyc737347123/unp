# TCP/UDP/IP学习笔记

## IP概览

### 1.0 IP地址分类

![](https://static001.geekbang.org/resource/image/0b/9e/0b32d6e35ff0bbc5d46cfb87f6669d9e.jpg)
在网络地址中，至少在当时设计的时候，对于A、B、C类主要分两部分，前面一部分是网络号，后面一部分是主机号。这很好理解，大家都是六单元 1001 号，我是小区 A 的六单元 1001 号，而你是小区 B 的六单元 1001 号。

下面这个表格，详细地展示了 A、B、C 三类地址所能包含的主机的数量
![](https://static001.geekbang.org/resource/image/e9/be/e9c59a4b2f0b804356759b10440ea7be.jpg)

### 1.1 CIDR

但是按 A、B、C 三类地址划分网络有不合适的地方，C类地址只能包含254个，现在一个网吧的机器都不只254台。而B类地址能包含的机器数量又太多，6万多台机器放在一个网络下，一般企业基本达不到这个规模，空闲的地址就是浪费

所以有了一个折中的方式叫**无类型域间选路**，简称**CIDR**。这种方式打破了原来设计的几类地址的做法，将 32 位的 IP地址一分为二，前面是网络号，后面是主机号。从哪里分呢？如果注意观察的话可以看到10.100.122.2/24，这个IP 地址中有一个斜杠，斜杠后面有个数字 24。这种地址表示表示形式，就是 CIDR。后面 24 的意思是，32 位中，前 24 位是网络号，后 8 位是主机号。

伴随着 CIDR 存在的，一个是**广播地址**，10.100.122.255。广播地址就是主机号二进制全为1，这样的地址就是这个网段的广播地址。如果发送这个地址，所有 10.100.122 网络里面的机器都可以收到。

另一个是**子网掩码**，255.255.255.0。将子网掩码和 IP 地址进行 AND 计算。前面三个 255，转成二进制都是 1。1 和任何数值取 AND，都是原来数值，因而前三个数不变，为 10.100.122。后面一个 0，转换成二进制是 0，0 和任何数值取 AND，都是 0，因而最后一个数变为 0，合起来就是 10.100.122.0。这就是网络号。**将子网掩码和 IP 地址按位计算 AND，就可得到网络号。**

### 1.2 公有IP地址和私有IP地址

私有地址也是分A、B、C三类的，在组网时具体用哪类的私有地址要根据局域网的主机数量来选择

私有地址是不需要申请就可以使用的，公有地址必须向**Inter NIC**申请。

公有IP地址在全球是唯一的，私有IP地址不是。我们和互联网上的其他主机通信时，用的都是公有IP地址，但是，在局域网中的机器只配置了私有IP地址，没关系，有机制可以把私有地址转换为公有IP地址，该机制就是**NAT（网络地址转换Network Address Translation）**

**私有地址就是为解决在IPv4下IP地址不够用而产生的**

### 1.3 IP的配置

#### 静态配置

Linux可以使用命令行配置ip，net-tools（即将弃用）和iproute2（目前的主流）都可以，但真正配置的时候，一定不是直接用命令行配置的，而是放在一个配置文件里面。**不同系统的配置文件格式不同，但是无非就是CIDR、子网掩码、广播地址、网关地址。**

#### 动态主机配置协议(DHCP)

动态主机配置协议(Dynamic Host Configuration Protocol)，简称DHCP。使用这个协议，网络管理员只需要配置一段共享的IP地址，每台新接入机器都通过DHCP协议，来这个共享的IP网段里申请IP。

打个比喻，**如果是数据中心的机器，IP一旦配置好，基本不会变，相当于买房自己装修(配置CIDR、网关地址等信息)。DHCP的方式相当于租房，不用装修，都帮你配置好了，你暂时用一下，用完退租就可以了。**

#### DHCP工作流程

1. **DHCP Discover**

   当一台机器新加入一个网络，新机器会使用IP地址0.0.0.0发送一个广播包，目的IP地址为255.255.255.255。广播包封装了UDP，UDP封装了BOOTP。其实DHCP是BOOTP的增强版，但是如果你去抓包，很可能看到的名称还是BOOTP
![](https://static001.geekbang.org/resource/image/39/1f/395b304f49559034af34c882bd86f11f.jpg)

2. **DHCP Offer**

   如果网络管理员在网络里面配置了**DHCP Server**，服务收到新机器的广播包后，根据MAC地址判断机器是否新机器，进而租给这个新机器一个IP地址，这个过程就是**DHCP Offer**。同时DHCP Server为此客户保留为它提供的IP地址，保证不会为其他DHCP客户分配此IP地址
![](https://static001.geekbang.org/resource/image/54/86/54ffefbe4f493f0f4a39f45504bd5086.jpg)
   
   DHCP Server仍然使用广播地址作为目的地址，因为此时请求分配IP的新机器还没有获得IP。
   
3. **DHCP Request**
  
   如果有多个DHCP Server，这个新机器会收到多个IP地址，它会选择其中一个DHCP Offer，一般是最先到达的那个，并且会向网络发送一个DHCP Request广播数据包，数据包的数据部分包含客户端的MAC地址，接受的租约中的IP地址，提供IP的DHCP Server的IP地址等。此时由于还没有得到DHCP Server的最后确认，客户端仍然使用0.0.0.0作为源IP，255.255.255.255作为目标地址进行广播
![](https://static001.geekbang.org/resource/image/e1/24/e1e45ba0d86d2774ec80a1d86f87b724.jpg)
   
4. **DHCP ACK**

   当DHCP Server收到客户机的DHCP Request后，会广播返回给客户机一个DHCP ACK数据包，表示已经接受客户机的选择，并将这个IP地址的合法租用信息和其他配置信息都放入包中，发给客户机
![](https://static001.geekbang.org/resource/image/7d/0e/7da571c18b974582a9cfe4718c5dea0e.jpg)

### 1.4 IP地址的收回和续租

既然是租约，那就有租期。租期到了，管理员就要将IP收回

如果想续租，就要提前一段时间通知DHCP Server

客户机会在租期过去50%的时候，直接向为其提供IP地址的DHCP Server发送DHCP Request消息包。客户机接受到改服务器回应的DHCP ACK消息包，会根据包中所提供的新租期以及其他已经更新的TCP/IP参数，更新自己的配置。这样IP租用更新就完成了

### 参考文献
- 极客时间<趣谈网络协议>
- [解析私有IP地址和公网IP地址](https://www.cnblogs.com/wbxjiayou/p/5150710.html)