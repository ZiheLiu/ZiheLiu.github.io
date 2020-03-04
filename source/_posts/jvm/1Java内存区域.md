---
title: 1. Java内存区域
date: 2020-03-04 14:48:52
tags:
	- java
	- jvm
categories:
	- java
	- jvm
typora-root-url: ../../
---

# 1. Java内存区域

## 0. 引用

- 《深入理解 Java 虚拟机 - 第三版》

## 0. 简介

JVM 在执行 Java 程序过程中，所管理的内存区域如下图，共有两类：

1. 由所有线程共享的数据区。随虚拟机进程的启动而一直存在。

2. 线程隔离的数据区。依赖用户线程的启动和结束而建立和销毁。

![JVM内存运行时数据区](/images/JVM内存运行时数据区.svg)

## 1. 程序计数器

当前线程所执行的字节码指令的行号指示器。

较小的内存空间。

- 如果线程正在执行的是 Java 方法，它记录正在执行的虚拟机字节码指令的地址。
- 如果线程正在执行的是本地 native 方法，它的值为空（undefined）。

## 2. Java 虚拟机栈

每个 Java 方法被执行时，JVM 会创建一个栈帧，存储局部变量表、操作数栈、动态连结、方法出口等信息。

方法从被调用到执行完毕的过程，对应栈帧在虚拟机栈中从入栈到出栈的过程。

### 局部变量表

局部变量表存放了

- 基本数据类型
- 对象引用
- returnAddress 类型（指向一条字节码指令的地址）

它的存储空间使用局部变量槽（Slot）来表示，64 位的 `long` `double` 占用两个变量槽。

局部变量表所需的内存空间，在编译期间就完成了分配。

### 异常

分配的栈过多、或栈的内存使用太多，会导致新的栈帧内存无法分配时，HotSpot 虚拟机都会抛出 `StackOverflowError`。

### JVM 参数

- -Xss：栈容量。
- -Xoss：本地方法栈容量。HotSpot 虚拟机不区分两个栈，所以这个参数没有用。

## 3. 本地方法栈

与 Java 虚拟机栈类似，为虚拟机使用到的本地 native 方法服务。

《Java 虚拟机规范》没有强制规定如何实现本地方法栈。HotSpot 虚拟机中，直接把 Java 虚拟机栈 和 本地方法栈合二为一。

## 4. Java 堆

唯一目的：存放对象实例。是垃圾收集器管理的内存区域。

### 对象的创建

1. 执行字节码 `new` 指令。
   1. 类加载检查。
      1. 检查指令的参数能否在常量池中定位到类的符号引用。
      2. 进行类的加载、解析、初始化。（如果没有进行的话）。
   2. 为对象分配内存。
      - 大小在类加载完成后可以完全确定。
      - 如果保证多个对象分配的线程安全？
        - CAS + 重试保证原子性更新。
        - 本地线程分配缓冲（Thread local Allocation Buffer，TLAB）。每个线程在 Java 堆中预先分配一块内存。分配内存现在线程的 TLAB 上进行，只有用完了，才去锁定。
   3. 将分配来的内存空间（除了对象头）初始化为零。
   4. 在对象头中设置必要的信息。
2. 执行 `<init>()` 方法，即构造函数。

### 对象的内存布局

1. 对象头部信息。
   1. Mark Word。对象自身运行时信息。对象哈希码、对象分代年龄、锁的信息。
   2. 类型指针。指向它的元数据的指针。
   3. 如果是 Java 数组，还需记录数组长度。
2. 实例数据部分。
3. 对齐填充。
   1. HotSpot 虚拟机要求对象起始地址必须是 8 字节的整数倍。

### 对象的访问定位

两种主流方式。

- 句柄。Java 堆中分出一块内存作为句柄池。reference 存储的是对象的句柄地址。句柄包含了对象实例数据的地址、类型数据的地址。
  - 这种方式，不需要对象头存储类型指针。
- 直接指针。reference 直接指向对象实例数据的地址，类型数据的地址存储在对象头的类型指针中。HotSpot 使用这种方式。

### 异常

`OutOfMemoryError` 。

### JVM 参数

- 堆的最小值：-Xms
- 堆的最大值：-Xmx

## 5. 方法区

存储被虚拟机加载的类型信息、常量、静态变量、即时编译器编译后的代码缓存等。

### 运行时常量池

包含

- .class 文件中包含 编译时生成的字面量、符号引用。
- 运行期间放入的新的常量，例如 `String.intern()`。

### 从永久代到元空间

在 JDK 7 以前，HotSpot 虚拟机使用永久代来实现方法区。把收集器的分代设计扩展到了方法区。

在 JDK 7 把字符串常量池、静态变量移到堆中。

在 JDK 8 把剩余内容（主要是类型信息）移到本地 native 内存的元空间中。

### 异常

`OutOfMemoryError`。

### JVM 参数

永久代大小（只对JDK 6、JDK 7中除字符串常量池、静态变量部分有效）

- -XX:PermSize 和 -XX:MaxPermSize。限制永久代的大小。

元空间大小（只对JDK 8中类型信息）

- -XX:MaxMetaspaceSize。最大值。
- -XX:MinMetaspaceSize。最小值。
- -XX:MetaspaceSize。初始大小。

## 6. 直接内存

不是 《Java 虚拟机规范》 定义的内存区域。

使用 native 函数 `unsafe.allocateMemory()` 分配的内存，直接在堆外分配。

通过 Java 堆里的 `DirectByeteBuffer` 作为这块内存的引用进行操作。

### 异常

- 使用  `ByeteBuffer.allocateDirect()` ，会抛出 `OutOfMemoryError:Direct buffer memory`。是在 Java 代码中，在向操作系统申请内存前，手动检查的。

- 使用 `unsafe.allocateMemory()`，会抛出 `java.lang.OutOfMemoryError\ n t sun.misc.Unsafe.allocateMemory(Native Method)`。是在操作系统申请内存时，抛出的。

### JVM 参数

- -XX:MaxDirectMemorySize。只在 `ByeteBuffer.allocateDirect()` 手动检查时使用。`unsafe` 分配不受它的限制。

