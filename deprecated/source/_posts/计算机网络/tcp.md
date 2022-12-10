---
title: TCP 协议
date: 2020-03-05 17:55:00
categories:
	- network
tags:
	- network
typora-root-url: ../
---

# 引用

- [TCP短连接和长连接的区别](https://zhuanlan.zhihu.com/p/47074758)
- [为什么基于TCP的应用需要心跳包（TCP keep-alive原理分析）](http://hengyunabc.github.io/why-we-need-heartbeat/)
- [聊聊 TCP 中的 KeepAlive 机制](https://zhuanlan.zhihu.com/p/28894266)
- [TCP新手误区--心跳的意义](https://blog.csdn.net/bjrxyz/article/details/71076442)

# 长连接、短连接

在TCP连接建立后，发送 一次数据后，就关闭连接。一般是由client关闭的。

建立连接后，双方不主动关闭连接。要使用keepalive来保活，活着使用应用层的心跳机制。

## 长连接的优点

1. 长连接可以省去较多的TCP建立和关闭的操作，减少网络阻塞的影响，

2. 当发生错误时，可以在不关闭连接的情况下进行提示，

3. 减少CPU及内存的使用，因为不需要经常的建立及关闭连接。

## 长连接的缺点

1. 连接数过多时，影响服务端的性能和并发数量。
2. 这时候server端需要采取一些策略，
   - 关闭一些长时间没有读写事件发生的连接，这样可以避免一些恶意连接导致server端服务受损。
   - 以客户端机器为颗粒度，限制每个客户端的最大长连接数。

# keepalive

KeepAlive并不是TCP协议规范的一部分，但在几乎所有的TCP/IP协议栈（不管是Linux还是Windows）中，都实现了KeepAlive功能。

KeepAlive机制默认是关闭的，需要将setsockopt将SOL_SOCKET.SO_KEEPALIVE设置为1才是打开KeepAlive状态。

流程：

1. 一端在一定时间（`tcp_keepalive_time`）内没有收到另外一端的请求后，会发出数据长度为0的心跳包。

2. 如果没有收到回复ack，那么隔一段时间（`tcp_keepalive_intvl`）后重试。

3. 如果重试`tcp_keepalive_probes`次后，还是没有收到ack。则主动断开连接。

默认值：

-  `tcp_keepalive_time`：2h
-  `tcp_keepalive_intvl`：75s
-  `tcp_keepalive_probes`：9次

设置keepalive的值

- 注意，这三个值是在操作系统里进行设置的，无法对某一个应用指定。

双向keepalive

- 哪一端想要探测对方端是否挂了，就需要开启keepalive。

和应用层自己实现的心跳的对比：

缺点

1. 它不是 TCP 的标准协议, 并且是默认关闭的，且经过路由等中转设备 keepalive 包可能会被丢弃。
2. 协议分层，各层关注点不同。
   - 传输层关注是否“通”。
   - 应用层关注是否可服务。
3. TCP的keepalive的数据长度为0，无法自定义；上层的心跳包可以自定义，携带更多的数据。
4. TCP 层的 keepalive 时间太长。
   - 默认 > 2 小时，虽然可改，但属于系统参数，改动影响所有应用。

缺点

- TCP的keepalive的数据包更小（数据长度为0）。

与HTTP中Connection: keep-alive的对比：

1. HTTP中Connection: keep-alive是说要复用这个TCP连接。
2. TCP中的keepalive是用探测机制看对方是否还活着。

# 三次握手与四次挥手

> [计算机网络 - 传输层](https://cyc2018.github.io/CS-Notes/#/notes/%E8%AE%A1%E7%AE%97%E6%9C%BA%E7%BD%91%E7%BB%9C%20-%20%E4%BC%A0%E8%BE%93%E5%B1%82?id=tcp-%e7%9a%84%e4%b8%89%e6%ac%a1%e6%8f%a1%e6%89%8b)

四次挥手中，第三次通信 B 向 A 发送关闭确认后，为什么 A 要 向 B 进行第四次通信？

- B 要确保 A 关闭了才可以。B 如果收不到 A 的第四次通信，会进行重试。

# 滑动窗口

> - [TCP/IP 系列之 TCP 流控与拥塞控制](http://mrpeak.cn/blog/tcp-flow-control00/)
>
> - [计算机网络 - 传输层](https://cyc2018.github.io/CS-Notes/#/notes/%E8%AE%A1%E7%AE%97%E6%9C%BA%E7%BD%91%E7%BB%9C%20-%20%E4%BC%A0%E8%BE%93%E5%B1%82?id=tcp-%e6%bb%91%e5%8a%a8%e7%aa%97%e5%8f%a3)

# 流量控制

流量控制是为了控制发送方发送速率，保证接收方来得及接收

发送窗口大小。

# 拥塞避免

- 拥塞窗口

- 慢开始与拥塞避免

- 快重传与快恢复

# UDP最大长度

最好不超过链路层的MTU(最大传输单元) 1500字节