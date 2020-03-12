---
title: Netty 记录
date: 2020-03-08 23:56:00
categories:
	- netty
tags:
	- netty
	- record
typora-root-url: ../../
---


# 为什么不直接用 JDK NIO

## Netty 做的更多

- 支持常用应用层协议。
- 解决传输问题：粘包、板报现象。
- 支持流量整形。
- 完善的断连、Idle 等异常处理。

## Netty 做的更好

- 规避 JDK NIO bug。
  - epoll bug：异常唤醒空转导致 CPU 100%。
  - IO_TOS 参数（IP 包的优先级和 QoS 选项）使用时抛出异常。
- API 更友好、更强大。
  - ByteBuffer 读写切换、无法扩容 -> ByteBuf。
  - ThreadLocal -> FastThreadLocal。
- 隔离变化、屏蔽细节。

# 为什么只支持 NIO

## 为什么不建议阻塞 I/O

- 连接数少时，并发度低，BIO 性能不差于 NIO。

- 连接数高时，阻塞->耗资源、效率低。

## 为什么不再支持 AIO

Linux 中 AIO 实现不成熟。相比较 NIO 性能提升不明显。

# 为什么 Netty 自己实现了 Epoll

- Netty 暴露了更多的可控参数。
  - JDK 的 NIO 只有水平触发
  - Netty 默认是边缘触发、可切换为水平触发。
- Netty 实现的垃圾回收更少、性能搞好。

见 {% post_link netty/epoll的et与lt %}

# Netty 与 Reactor 模式

## MainReactor 一般单线程

Netty 的 MainReactor，每次 `bind(address)` 会注册一次 `ServerSocketChannel` 到 `EventLoopGroup` 上，只能使用一个线程。

## 分配 EventLoop 规则

为 Channel 分配 EventLoop 时，循环选择下一个。线程池数为 2 的次幂时，则用位运算优化。

## 通用模式的 NIO 实现多路复用器跨平台

`DefaultSelectorProvider.create()` 不同操作系统会创建不同的 `SelectorProvider`。

```java
package sun.nio.ch;
// MaxOS
public class DefaultSelectorProvider {
    public static SelectorProvider create() {
        return new sun.nio.ch.KQueueSelectorProvider();
    }
}
// Windows
public class DefaultSelectorProvider {
    public static SelectorProvider create() {
        return new WindowsSelectorProvider();
    }
}
```

# 粘包与半包

## 粘包原因

- 发送方每次写入数据 < 套接字缓冲区大小。

- 接收方读取套接字缓冲区数据不够及时。

## 半包原因

- 发送方写入数据 > 套接字缓冲距大小。
- 发送的数据大于协议的 MTU（Maximum Transmission Unit，最大传输单元），必须拆包。

## 根本原因

TCP 是流式协议，消息无边界。

## UDP

UDP 像邮寄的包裹，虽然一次运输多个，但每个包裹都有“界限”，一个一个签收， 所以无粘包、半包问题。

## 如何解决

找出消息的边界：封装成帧。

- 固定长度。浪费空间。`FixedLengthFramDecoder`。
- 分隔符。内容本身出现分隔符时需要转移。`DelimiterBasedFrameDecoder`。
- 固定字节数size存长度信息。长度受字节数size限制，需要提前预知可能的最大长度。`LengthFieldBasedFrameDecoder`解码 与 `LengthFieldPrepender` 编码。

### ByteToMessageDecoder

需要积累数据。两种方式：

1. 拷贝到一个 `ByteBuf` 里。

2. 使用 `CompositeByteBuf` 视图组装多个 `ByteBuf`。

但是第 2 种方式使用了更为复杂的索引方式，根据使用场景、实现解码方式的不同，可能效率比第一种低。

扩容发生在：

- 第 1 种方式，`cumulation` 放不下下一个 `inByteBuf` 了、或 `cumulation` 被调用了 `retain() ` 或者是只读的。
- 第 2 种方式，`cumulation` 被调用了 `retain() `。

# 二次解码器

- 一次解码器：`ByteToMessageDecoder`
  - io.netty.buffer.ByteBuf （原始数据流）-> io.netty.buffer.ByteBuf （用户数据）

- 二次解码器：`MessageToMessageDecoder<I>`
  - io.netty.buffer.ByteBuf （用户数据）-> Java Object

## 对 JDK Serialization 的支持

Netty 进行了优化。

- 解码后，压缩。
- JDK 写入了每个变量的类型、所有修饰符。Netty 只写入了名字，运行时通过反射找到它的其他信息。

## 对 protobuf 的支持

- 一次解码时，没有使用`LengthFieldBasedFrameDecoder`，而是进行了优化。
  - 记录长度的字节数目是可变的，`VarInt`。

- 二次解码时，把一次解码得到的字节数组，根据 protobuf 的规则解析为对象。

# KeepAlive

## 开启 TCP keepalive

在 Server 端

```java
// 方式一
bootstrap.childOption(ChannelOption.SO_KEEPALIVE,true) 
// 方式二  
bootstrap.childOption(NioChannelOption.of(StandardSocketOptions.SO_KEEPALIVE), true)
```

## 应用层 KeepAlive

### 方法一 定时发送

每隔一段时间，分别向所有客户端发送一个 keepalive 消息。

耗费资源，如果客户端太多。

### 方法二 空闲 Idle 检测

无数据传输超过一段时间，判定为 Idle，再发 keepalive 消息。

快速释放损坏的、恶意的、很久不用的连接。

## Idle 检测

```java
ch.pipeline().addLast("idleCheckHandler", new IdleStateHandler(0, 20, 0, TimeUnit.SECONDS));
```

  ### ReaderIdle

1. 在 `channelActive()` 中注册延时（`readerIdleTimeNanos` 纳秒） 任务。
2. 在 `channelRead()` 中把 `reading` 设为 true。
3. 在 `channelReadComplete()` 中把 `reading` 设为 false，记录读完的时间。
4. 延时时间到了，执行检测任务，如果没有正在发生读事件（`reading` 为 false）、且距离上次读完成超过了 `readerIdleTimeNanos ` 纳秒，就在pipeline上传递一个 `READER_IDLE` 事件。
5. 重写注册延时任务。

### WriterIdle

1. 在 `channelActive()` 中注册延时（`writerIdleTimeNanos` 纳秒） 任务。
2. 延时时间到了，执行检测任务。
   - 如果距离上次写完成超过了 `writerIdleTimeNanos` 纳秒，就在pipeline上传递一个 `WRITER_IDLE` 事件。
   - 如果开启了 `observeOutput`，则只要有写意图（还没有写完），就不会认为是 Idle 状态。
3. 重新注册延时任务。

### 使用例子

#### 目的

- 客户端隔一段**空闲**时间向服务端发送 HEART_BEAT 消息。

- 服务端隔一段**空闲**时间没有收到客户端的 HEAR_BEAT 消息，就关闭这个连接。

#### 实现

- 在客户端开启一个 WriterIdle 检测，超过给定时间后，发送 HEART_BEAT 消息给服务端。
- 在服务端开启一个 ReaderIdle 检测，超过给定时间没有收到消息后，关闭连接。
- 客户端在 `channelInactive()` 中，尝试重连服务端。

# Netty 中锁的使用

## 注意锁的对象和范围 减小粒度

Synchronized method -> Synchronized block

![netty锁的粒度](/images/netty锁的粒度.png)

## 注意锁的对象本身大小 减少空间占用

- volatile long = 8 bytes

- AtomicLong = 8 bytes （volatile long）+ 16bytes （对象头）+ 8 bytes (引用) = 32 bytes

## 注意锁的速度 提高并发性

结论： 及时衡量、使用 JDK 最新的功能

![nettyLongCounter](/images/nettyLongCounter.png)

## 不同场景选择不同的并发包 因需而变

`NioEventloop` 中负责存储 task 的 Queue，单线程的，只有一个消费者。

JDK LinkedBlockingQueue (MPMC) -> jctools’ MPSC

单消费者：出队不用加锁。

## 衡量好锁的价值 尽量不用锁

### EventLoop

Netty 使用局部串行 + 整体并行，而不是 一个队列 + 多个线程模式。

- 一个 Channel 只由一个 EventLoop（一个线程）负责。

- 一个 EventLoop 负责多个 Channel。

优点

- 降低用户开发难度、逻辑简单、提升处理性能。

- 避免锁带来的上下文切换、并发保护等额外开销。

### Recycler

用 ThreadLocal 来避免资源争用，例如 Netty 轻量级的线程池实现

# Netty 内存使用技巧

## 减少对像本身大小

1. 用基本类型就不要用包装类。
2. 应该定义成类变量的不要定义为实例变量。

volatile long -> AtomicLong

## 预测分配大小

根据接受到的数据动态调整（guess）下个要分配的 Buffer 的大小。

## 零拷贝

- 使用逻辑组合，代替实际复制。
  - CompositeByteBuf
- 使用包装，代替实际复制。
  - ByteBuf byteBuf = Unpooled.wrappedBuffer(bytes)。
- 调用 JDK 的 Zero-Copy 接口。
  - DefaultFileRegion 中包装了 NIO 的 FileChannel.transferTo()
- 对外内存。

## 内存池

### 为什么引入对象池

- 创建对象开销大。

- 对象高频率创建且可复用。

- 支持并发、又能保护系统。

- 维护、共享有限的资源。

### Recycler

见 {% post_link netty/Recycler对象池 %}

