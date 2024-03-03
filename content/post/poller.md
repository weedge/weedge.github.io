---
author: "weedge"
title: "网络模型"
date: 2021-09-02T10:26:23+08:00
tags: [
	"golang","网络编程"
]
categories: [
	"技术",
]

---

看了一些开源的网络I/O模型框架库，尝试着按照理解简单实现一个相对简单的网络I/O模型框架，类似netty的reactor模型。

<!--more-->

Netty的NIO模型是Reactor反应堆模型（Reactor相当于有分发功能的多路复用器Selector）。每一个连接对应一个Channel（多路指多个Channel，复用指多个连接复用了一个线程或少量线程，在Netty指EventLoop），一个Channel对应唯一的ChannelPipeline，多个Handler串行的加入到Pipeline中，每个Handler关联唯一的ChannelHandlerContext。

Reactor 模式的基本工作流程如下：

- Server 端完成在 `bind&listen` 之后，将 listenfd 注册到 epollfd 中，最后进入 event-loop 事件循环。循环过程中会调用 `select/poll/epoll_wait` 阻塞等待，若有在 listenfd 上的新连接事件则解除阻塞返回，并调用 `socket.accept` 接收新连接 connfd，并将 connfd 加入到 epollfd 的 I/O 复用（监听）队列。
- 当 connfd 上发生可读/可写事件也会解除 `select/poll/epoll_wait` 的阻塞等待，然后进行 I/O 读写操作，这里读写 I/O 都是非阻塞 I/O，这样才不会阻塞 event-loop 的下一个循环。然而，这样容易割裂业务逻辑，不易理解和维护。
- 调用 `read` 读取数据之后进行解码并放入队列中，等待工作线程处理。
- 工作线程处理完数据之后，返回到 event-loop 线程，由这个线程负责调用 `write` 把数据写回 client。



参考这些网络模型，采用golang封装的底层epoll/kqueue系统调用方法，支持tcp协议，实现一个相对简单的网络模型，框架如下：

![go-epoll.png](https://raw.githubusercontent.com/weedge/im/main/go-epoll.png)

代码实现：[https://github.com/weedge/lib/tree/main/poller](https://github.com/weedge/lib/tree/main/poller)  (对一个开源库进行的改造，codec编解码器待完善)

#### 参考：

1. [gnet](https://github.com/panjf2000/gnet)
2. [netpoll](https://github.com/cloudwego/netpoll)
3. [evio](https://github.com/tidwall/evio)
4. [getty](https://github.com/AlexStocks/getty)
5. [netty](https://github.com/netty/netty)
6. [Linux网络编程模型](https://github.com/xinali/articles/issues/57)