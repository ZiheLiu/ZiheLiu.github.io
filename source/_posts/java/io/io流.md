---
title: I/O流相关类
date: 2020-03-04 14:48:52
tags:
	- java
	- io
categories:
	- java
	- io
---

# I/O 流相关类

## 0. 引用

- [Java IO，硬骨头也能变软](https://zhuanlan.zhihu.com/p/28286559)
- [PipedInputStream和PipedOutStream记录](https://zhuanlan.zhihu.com/p/31352578)
- 《Java 核心技术-卷2-第二章》

## 1. 三种分类方式

### 字节流和字符流

- 字节流
  - 以字节为单位进行读写。
  - 每次读写 1 byte 数据。
  - 可以操作任何类型的数据。
- 字符流
  - 以字符码元为单位进行读写。
  - 每次读写 2 bytes 数据。
  - 只能操作字符类型的数据。

### 输入流和输出流

- 输入流
  - 可以进行输入相关的操作。
- 输出流
  - 可以进行输出相关的操作。

### 节点流和操作流

- 节点流
  - 直接与数据相连，数据来源或目的地是要操作的最终数据。
- 操作流
  - 利用装饰器模式，对节点流进行包装，提供多种方便的做操方式。

## 2. 具体分类

所有读取/写入操作都要加锁。

### 2.1 输入/字节流

#### 1) 节点流

##### FileInputStream

- 用于从文件读取数据。
- 只能读取字节或字节数组。

##### ByteArrayInputStream

- 用于从数组读取数据。
- 只能读取字节或字节数组。

##### PipedInputStream

- 用于线程间通信，需要和 `PipedOutputStream` 一起使用。
- 使用固定长度的字节数组 buffer，在没有可读字节时阻塞。
- 只能读取字节或字节数组。

#### 2) 操作流

##### FilterInputStream

- 内置一个输入/字节流 inner 用来操作。
- 所有操作只是简单的调用 inner 来完成。
- 所有输入/字节/操作流的超类，除了 SequenceInputStream。

##### BufferedInputStream

- 使用字节数组 buffer 缓存数据。每次读取，先把 buffer 填满，再从 buffer 中读取。
- 只能读取字节或字节数组。

##### DataInputStream

- 实现了 `DataInput` 接口。
  - `readInt/Char/Boolean/等基本类型` 读取基本类型
  - `readUTF` 读取 修订过的 UTF 格式的字符串。
    - 使用 `readUTF` 读取字符串。
    - 2个字节的 uint 表示长度 length。
    - length 个字节组成 修订过的 UTF 编码。

##### ObjectInputStream

- 用于序列化对象。
- 实现了 `DataInput` 接口。
- `readObject` 读取对象。

##### SequenceInputStream

- 从多个流依次读取数据。
- 构造函数传入任意个 InputStream。

### 2.2 输出/字节流

#### 1) 字节流

##### FileOutputStream

- 用于向文件写入数据。
- 只能写入字节或字节数组。

##### ByteArrayOutputStream

- 用于向数组写入数据。
- 只能写入字节或字节数组。

##### PipedOutputStream

- 用于线程间通信，需要和 `PipedInputputStream` 一起使用。
- 在没有空间写入时阻塞。
- 只能写入字节或字节数组。

#### 2) 操作流

##### FilterOutputStream

- 内置一个输出/字节流 inner 用来操作。
- 所有操作只是简单的调用 inner 来完成。
- 所有输出/字节/操作流的超类。

##### BufferedOutputStream

- 使用字节数组 buffer （默认 1024 字节）缓存数据。每次写入，先写入 buffer，buffer 满了再写入流。
- 只能写入字节或字节数组。

##### DataOutputStream

- 实现了 `DataOutput` 接口。
  - `writeInt/Char/Boolean/等基本类型` 写入基本类型
  - `writeUTF` 写入 修订过的 UTF 格式的字符串。
  - `writeChars` 写入 UTF16 格式的字符串。注意没有写入长度。

##### ObjectOutputStream

- 序列化对象。
- `writeObject` 读取对象。

##### PrintStream

- 用于打印基本类型/字符串到流。
  - 都转化为字符串。
- 性能不好。
  - 内部使用 `BufferedWriter`。
  - 每个 write 操作后，会 flushBuffer，把 buffer 的内容写入到 inner 流中（没有 flush inner）。
  - `prinln` 分成两次 write 操作：writer 内容 + writer 换行符。要 flushBuffer 两次。
  - 开启 autoflush 后，每次写入换行符，都会调用 inner 的 flush 方法。

### 2.3 输入/字符流

除了 `BufferedReader.readLine()`，都只能读取字符/字符数组。

#### 1) 节点流

##### FileReader

- 用于从文件读取数据。
- 继承了 InputStreamReader。创建一个 fileInputStream，传递给自己。
- 只能读取字符/字符数组。

##### PipedReader

- 用于线程间通信。
- 使用字符数组缓存数据。没有字符可读时，阻塞。
- 只能读取字符/字符数组。

##### CharArrayReader

- 用于从数组读取数据。
- 只能读取字符/字符数组。

#### 2) 操作流

##### BufferedReader

- 只能读取字符/字符数组。
- 使用字符数组 buffer （默认 8192 字节）缓存数据。
- 读取一行字符串。

##### InputStreamWriter

- 把 输入/字节流 转化为 输入/字符流。
- 只能读取字符/字符数组。
- 需要给定编码格式。如果不给定，则为 JVM 的默认字符集，和操作系统有关。

### 2.4 输出/字符流

#### 1) 节点流

##### FileWriter

- 用于向文件写入数据。
- 继承了 OutputStreamReader。创建一个 fileOutputStream，传递给自己。
- 只能写入字符/字符串/字符数组。

##### PipedWriter

- 用于线程间通信。
- 没有空间写入时，阻塞。
- 只能写入字符/字符串/字符数组。

##### CharArrayWriter

- 用于向数组写入数据。
- 只能写入字符/字符串/字符数组。

#### 2) 操作流

##### BufferedWriter

- 只能写入字符/字符串/字符数组。
- 使用字符数组 buffer （默认 8192 字节）缓存数据。每次写入，先写入 buffer，buffer 满了再写入流。

##### OutputStreamWriter

- 把 输出/字节流 转化为 输出/字符流。
- 只能写入字符/字符串/字符数组。
- 需要给定编码格式。如果不给定，则为 JVM 的默认字符集，和操作系统有关。

##### PrintWriter

- 用于打印基本类型/字符串到流。
- 性能不好。
  - 每个 writer 操作，都会写入 inner 流中。
  - `println` 分开两次 write 操作。
  - 开启 autoflush 后，每次写入后都会调用 inner 的 flush 方法。

## 3. 例子

使用 `connect`把 `PipedInputStream` 与 `PipedOutputStream`连接起来。

```java

class Sender implements Runnable {
  PipedOutputStream out = new PipedOutputStream();

  public PipedOutputStream getOut() {
    return out;
  }

  @Override
  public void run() {
    try {
      out.write("hello, world".getBytes(StandardCharsets.UTF_8));
    } catch (IOException e) {
      e.printStackTrace();
    }
  }
}

class Receiver implements Runnable {
  PipedInputStream in = new PipedInputStream();

  public PipedInputStream getIn() {
    return in;
  }

  @Override
  public void run() {
    try {
      byte[] buffer = new byte[100];
      in.read(buffer);
      System.out.println(new String(buffer, StandardCharsets.UTF_8));
    } catch (IOException e) {
      e.printStackTrace();
    }
  }
}

class Main {
  public static void main(String[] args) throws IOException {
    Sender sender = new Sender();
    Receiver receiver = new Receiver();

    sender.getOut().connect(receiver.getIn());

    new Thread(sender).start();
    new Thread(receiver).start();
  }
}
```

