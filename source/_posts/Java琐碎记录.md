---
title: Java琐碎记录
date: 2018-09-29 14:48:52
tags:
	- java
	- 琐碎记录
---
## IO

### 内存映射文件

传统的读取文件方式，要先把文件内容拷贝到内核的IO缓存区，再从内核的IO缓存区拷贝到进程的私有空间中。

而内存映射文件，可以直接把进程的用户私有空间的一部分区域与文件对象建立起映射关系，不需要拷贝到内核的IO缓存区。

在Java NIO中创建内存映射文件方式如下：

```java
File file = new File("filename");

// 读的时候，从FileInputStream或RandomAccessFile得到Channel都可以
FileInputStream fin = new FileInputStream(file);
FileChannel inChannel = fin.getChannel();
MappedByteBuffer inBuffer = inChannel.map(FileChannel.MapMode.READ_ONLY, 0, inChannel.size());

// 写的时候，只能从RandomAccessFile得到Channel
RandomAccessFile fout = new RandomAccessFile(file, "rw");
FileChannel outChannel = fout.getChannel();
MappedByteBuffer outBuffer = outChannel.map(FileChannel.MapMode.READ_ONLY, 0, outChannel.size());
```

