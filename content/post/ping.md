---
author: "weedge"
title: "工具盒子-ping/traceroute"
date: 2021-11-15T14:26:23+08:00
tags: [
	"ping",
]
categories: [
	"技术",
]


---

## 序言

在访问网络是否ok, 通常喜欢用ping 命令来访问`ping www.baidu.com` 看是否出现超时；ping在不同的操作系统平台实现方式差不多，底层都是用ICMP协议，每次发ICMP ECHO_REQUEST packet (IP地址/Host, ttl，icmp_seq序列号，[RTD/RTT](https://en.wikipedia.org/wiki/Round-trip_delay)(往返延时)记录 ),  运行结束后统计每个RTT, 最大RTT, 最小RTT, 平均RTT, [标准偏差](https://en.wikipedia.org/wiki/Algorithms_for_calculating_variance#Welford)RTT, 发送/接受packets总数，丢包率 等数据，ping工具在PING(8)中的定义如下：

> The ping utility uses the ICMP protocol's mandatory ECHO_REQUEST datagram to elicit an ICMP ECHO_RESPONSE from a host or gateway.  ECHO_REQUEST datagrams  ("pings'') have an IP and ICMP header, followed by a "struct timeval'' and then an arbitrary number of "pad" bytes used to fill out the packet.

<!--more-->

## [ICMP](https://en.wikipedia.org/wiki/Internet_Control_Message_Protocol)

**互联网控制消息协议**（英语：**I**nternet **C**ontrol **M**essage **P**rotocol，缩写：**ICMP**）是[互联网](https://zh.wikipedia.org/wiki/互联网)协议族的核心协议之一。它用于[网际协议](https://zh.wikipedia.org/wiki/网际协议)（IP）中发送控制消息，提供可能发生在通信环境中的各种问题反馈。通过这些信息，使管理者可以对所发生的问题作出诊断，然后采取适当的措施解决；使用在网络设备上，比如交换机，路由器

### 为毛ICMP设计在网络层呢？ 

因为需要目的地址ip, 链路层无法通过ip socket编程，那为啥不用链路层的MAC地址呢？因为链路层的MAC地址只在局域网唯一，由交换机学习缓存策略决定(会缓存MAC地址，用于跳转)，如果广域网访问不同局域网，MAC地址可以不唯一(网络历史演变原因)，但是公网IP地址是全局唯一，相互转化通过NAT路由器(具体介绍看wiki: **[Network address translation](https://en.wikipedia.org/wiki/Network_address_translation)**)； 如图所示：

![NAT](https://raw.githubusercontent.com/weedge/lib/main/net/NAT.png)

如果在传输层的话，无需端口多了一次解包；ICMP与[TCP](https://en.wikipedia.org/wiki/Transmission_Control_Protocol)、[UDP](https://en.wikipedia.org/wiki/User_Datagram_Protocol)等[传输协议](https://en.wikipedia.org/wiki/Transport_protocol)不同因为它通常不用于在系统之间交换数据，也不经常被最终用户网络应用程序使用（除了一些诊断工具，如[ping](https://en.wikipedia.org/wiki/Ping_(networking_utility))和[traceroute](https://en.wikipedia.org/wiki/Traceroute)）。

ICMP 依靠ip来完成它的任务，它是ip的主要部分。它与传输协议（如[TCP](https://zh.wikipedia.org/wiki/传输控制协议)和[UDP](https://zh.wikipedia.org/wiki/用户数据报协议)）显著不同：它一般不用于在两点间传输数据。它通常不由网络程序直接使用，除了 [ping](https://zh.wikipedia.org/wiki/Ping) 和 [traceroute](https://zh.wikipedia.org/wiki/Traceroute) 这两个特别的例子。 [IPv4](https://zh.wikipedia.org/wiki/IPv4)中的ICMP被称作ICMPv4，[IPv6](https://zh.wikipedia.org/wiki/IPv6)中的ICMP则被称作[ICMPv6](https://zh.wikipedia.org/wiki/ICMPv6)。

## [ping](https://en.wikipedia.org/wiki/Ping_(networking_utility))

Ping (Packet Internet Grope)，因特网包探索器，用于测试网络连接量的程序。Ping发送一个ICMP回声请求消息给目的地并报告是否收到所希望的ICMP回声应答；具体流程如下：

![ping-icmp](https://raw.githubusercontent.com/weedge/lib/main/net/ping-icmp.png)

**TTL**： 生存时间，指定<u>数据包被路由器丢弃之前允许通过的网段数量</u>；<u>**TTL 是由发送主机设置的，当其存活次数为0时，路由器便会取消数据包并发送一个ICMP TTL数据包给原数据包的发出者, 以防止数据包不断在 ip 互联网络上永不终止地循环, 而无法送达及耗尽网络资源**</u>。转发 ip 数据包时，要求路由器至少将 TTL 减小 1；TTL 字段值不同操作系统类型对应的值不同，可以通过`ping 127.0.0.1` 本地来查看初始值,或者`sysctl -a |grep ttl `查看对应值(macOS Darwin下是net.inet.ip.ttl, linux下是net.ipv4.ip_default_ttl)；如果不修改这些值，linux系统一般是64，window系统一般是128。

## [traceroute](https://en.wikipedia.org/wiki/Traceroute)

traceroute是unix系统中，诊断[计算机网络](https://en.wikipedia.org/wiki/Computer_network)的命令程序(windos中对应tracert命令)；用于显示可能的路线（路径）和测量的传送延迟。路由的历史记录为从路由（路径）中每个连续主机（远程节点）接收到的数据包的往返次数；每[跳](https://en.wikipedia.org/wiki/Hop_(networking))平均时间的总和是建立连接所花费的总时间的度量；如果所有（通常是三个）发送的数据包丢失两次以上，连接就会丢失并且无法评估路由，否则 Traceroute 会继续。

主叫方首先发出 TTL=1 的数据包，第一个路由器将 TTL 减1得0后就不再继续转发此数据包，而是返回一个 ICMP 超时报文，主叫方从超时报文中即可提取出数据包所经过的第一个网关地址。然后又发出一个 TTL=2 的 ICMP 数据包，可获得第二个网关地址，依次递增 TTL 便获取了沿途所有网关地址。

需要注意的是，并不是所有网关都会如实返回 ICMP 超时报文。出于安全性考虑，大多数防火墙以及启用了防火墙功能的路由器缺省配置为不返回各种 ICMP 报文，其余路由器或[交换机](https://zh.wikipedia.org/wiki/交换机)也可被管理员主动修改配置变为不返回 ICMP 报文。因此 Traceroute 程序不一定能拿全所有的沿途网关地址。所以，当某个 TTL 值的数据包得不到响应时，并不能停止这一追踪过程，程序仍然会把 TTL 递增而发出下一个数据包。一直达到默认或用参数指定的追踪限制（maximum_hops）才结束追踪（没有到达目标ip, 所以收不到目标ip的ICMP报文）。

而ping工具只计算从目的地点的最终往返时间。

当然ping/traceroute，不一定用网络层的ICMP协议来实现，也可以用传输层的[UDP协议](https://en.wikipedia.org/wiki/User_Datagram_Protocol)来实现(发送udp协议报文),默认主目的主机端口是33434开始)，这个端口是有讲究的；利用了 UDP 数据包的 traceroute 程序在数据包到达真正的目的主机时，就可能因为该主机没有提供 [UDP](https://zh.wikipedia.org/wiki/用户数据报协议) 服务而简单将数据包抛弃，并不返回任何信息。为了解决这个问题，traceroute 故意使用了一个大于 30000 的端口号(因 UDP 协议规定端口号必须小于 30000)，所以目标主机收到数据包后唯一能做的事就是返回一个“端口不可达(ICMP PORT_UNREACHABLE)”的 ICMP 报文，于是主叫方就将端口不可达报文当作跟踪结束的标志。

甚至traceroute还用[TCP协议](https://en.wikipedia.org/wiki/Transmission_Control_Protocol)来实现(发送tcp协议报文)，这个需要看场景下，需要获取的测试监控数据是哪些，是否有必要建立可靠连接。比如一些traceroute 实现使用TCP 数据包，例如*tcptraceroute*和[第四层traceroute](https://en.wikipedia.org/wiki/Layer_four_traceroute) (LFT)。

具体使用可以参考`man traceroute` 中的介绍，以上说的，在man文档里都有介绍到；

可以通过`traceroute -P ICMP  www.baidu.com` 使用ICMP协议来追踪每一跳的路由情况；支持ICMP, UDP(默认发送方式), TCP协议，具体见文档；

如果想查看ICMP协议包的内容可以通过[Wireshark/TShark(命令)](https://gitlab.com/wireshark/wireshark/-/wikis/home)来抓包，主要是通过**协议报文**，来分析网络问题；你会发现ping 命令程序只用到了IP/ICMP协议(send Type: 8 (Echo (ping) request)；recv Type: 0 (Echo (ping) reply) from dst ip)， 而traceroute 命令程序会用到IP/ICMP(send Type: 8 (Echo (ping) request)；recv Type: 11 (Time-to-live exceeded) code:0 from 每个路由，recv Type: 0 (Echo (ping) reply) from dst ip), IP/UDP协议(如果用UDP发送，send udp 数据报文, recv Type: 11 (Time-to-live exceeded) code:0 from  每个路由， recv Type: 3 (Destination unreachable) code:3 from dst ip)。

可以分析出，ping 和 traceroute 都使用IP/ICMP协议时，traceroute 是接收了每一跳路由的响应进行处理，然后展现的每一跳累计的总耗时。流程如下：

![traceroute-icmp](https://raw.githubusercontent.com/weedge/lib/main/net/traceroute-icmp.png)



## 轮子

为啥要用golang重新写一个工具呢？ （已有开源的 [go-ping](https://github.com/go-ping/ping.git), 支持icmp, udp (icmp.ListenPacket [x/net/icmp](https://godoc.org/golang.org/x/net/icmp)) 直接查看源码吧）

1. 平时虽然用，但是具体实现想了解下，主要是[ICMP协议](https://en.wikipedia.org/w/index.php?title=Internet_Control_Message_Protocol)中的控制信息([Control messages](https://en.wikipedia.org/w/index.php?title=Internet_Control_Message_Protocol#Control_messages)), 以便在次协议上DIY网络需求；
2. golang是跨系统平台编译语言，同一份代码编译运行在不同平台；
3. 可以做一些扩展，运用在K8S的编排容器中测试网络环境；因为icmp定义在网络层，只需ip，无需服务端口，利用icmp协议做一些扩展功能；比如机器是否挂了，目的ip是否不可到达了，以及做一些网络层监控等。

## reference

1. [PING](https://en.wikipedia.org/wiki/Ping_(networking_utility))
2. [ICMP](https://en.wikipedia.org/wiki/Internet_Control_Message_Protocol)
3. [TTL](https://www.techtarget.com/searchnetworking/definition/time-to-live)
4. [NAT](https://en.wikipedia.org/wiki/Network_address_translation)
5. [PING Command - Troubleshooting](https://www.youtube.com/watch?v=IIicPE38O-s) 

