---
title: Netty记录
date: 2020-03-05 14:33:57
categories:
	- netty
tags:
	- netty
---

# 引用

- [Netty 零拷贝（三）Netty 对零拷贝的改进](https://www.cnblogs.com/binarylei/p/10117437.html)

# Netty 中的零拷贝

1. Netty 接收和发送 `ByteBuffer` 使用 `DirectByteBuffer`。消除了从堆外内存拷贝到堆内内存。
2. 通过 `FileRegion` 包装的 `FileChannel#transferTo()` 方法实现文件传输。消除了内核与用户空间的拷贝。
3. `CompositeByteBuf` 类，将多个 `ByteBuf` 在逻辑上合并为一个 `ByteBuf`。消除了合并时的拷贝。
4. 通过 `wrap` 操作，将 `byte[] `、`ByteBuf`、`ByteBuffer` 等包装成一个 `ByteBuf`，避免了拷贝操作。

## FileRegion

在将文件写入 `channel` 时，可以使用 `FileRegion`。

```java
public class FileServerHandler extends SimpleChannelInboundHandler<String> {

    @Override
    public void channelRead0(ChannelHandlerContext ctx, String msg) throws Exception {
        RandomAccessFile raf = new RandomAccessFile(msg, "r");
     		ctx.write(new DefaultFileRegion(raf.getChannel(), 0, length));
        ctx.writeAndFlush("\n");
    }
}
```

查看 `DefaultFileRegion` 源码，可以看到写入操作是使用 `FileChannel#transferTo()` （见 {% post_link java/io/内存映射文件 FileChannel 与内存映射文件 %}）实现的。

```java
public class DefaultFileRegion extends AbstractReferenceCounted implements FileRegion {
  @Override
  public long transferTo(WritableByteChannel target, long position) throws IOException {
    long count = this.count - position;
    if (count < 0 || position < 0) {
      throw new IllegalArgumentException(
        "position out of range: " + position +
        " (expected: 0 - " + (this.count - 1) + ')');
    }
    if (count == 0) {
      return 0L;
    }
    if (refCnt() == 0) {
      throw new IllegalReferenceCountException(0);
    }
    // Call open to make sure fc is initialized. This is a no-oop if we called it before.
    open();
		// call FileChannel#transferTo。
    long written = file.transferTo(this.position + position, count, target);
    if (written > 0) {
      transferred += written;
    }
    return written;
  }
}
```

