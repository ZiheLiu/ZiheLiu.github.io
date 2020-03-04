---
title: 3. Java 内存模型
date: 2020-03-04 14:48:03
tags:
	- java
	- concurrency
categories:
	- java	
	- concurrency
typora-root-url: ../../
---

# 3. Java 内存模型

## 0. 引用

- [Java内存模型（JMM）总结](https://zhuanlan.zhihu.com/p/29881777)

## 1. Java 内存模型

### 1.1 定义

Java 内存模型（Java Memory Model（JMM））是虚拟机规范，用于屏蔽各种硬件和操作系统的差异，以使 Java 程序在各个平台达到一致的并发效果。

### 1.2 栈与堆

线程栈中的变量不能多线程访问。

堆中的变量可以多线程访问。

### 1.3 happens-before 规则

JSR-133 内存模型，使用 happens-before 规则来阐述操作之间的可见性。

#### 定义

1. 如果操作 A happens-before 操作 B，那么操作 A 的执行结果对操作 B 可见、且操作 A 的执行顺序排在操作 B 之前。
2. 两个操作之间存在 happens-before关系，并不意味着Java平台的具体实现必须要按照 happens-before关系指定的顺序来执行。
   如果重排序之后的执行结果，与按 happens-before 关系来执行的结果一致，这种就重排序并不非法。JMM允许这种重排序。
   例如下面程序，A happens before B，A 操作的结果不需要对 B 操作可见，并且重排 A 和 B 的顺序，执行结果不变。JMM 认为这种重排序是合法的。

  ```java
  double pi = 3.14; 				// A
  double r = 1.0;      			// B
  double area = pi * r * r;	// C
  ```

#### as-if-serial 语义

不管怎么重排序，**单线程**程序的**执行结果**不能被改变。

happens-before 本质上和 as-if-serial 语义一样。

#### 规则

具体规则如下：

- 程序顺序规则：一个线程中的每个操作，happens-before 于该线程后续操作。
- 监视器锁规则：对一个锁的解锁，happens-before 于随后对这个锁的解锁。
- `volatile` 变量规则：对一个 `volatile` 域的写，happens-before 于后续对这个 `volatile` 域的读。
- 传递性：如果 A happens-before B、B happens-before C，则 A happens-before C。
- `start()` 规则：如果线程 A 执行操作 `ThreadB.start()` 来启动线程 B，那么 A 线程的 `ThreadB.start()` 操作 happens-before 于线程 B 中的任意操作。
- `join()` 规则：如果线程 A 执行操作 `ThreadB.join()`并成功返回，那么线程B中的任意操作 happens-before 于线程 A 从 `ThreadB.join()` 成功返回的操作。

### 1.4 JMM 的内存可见性保证

1. 单线程程序。单线程程序不会出现内存可见性问题。保证单线程程序与在顺序一致性模型中的执行结果相同。
2. 正确同步的多线程程序。保证正确同步的多线程程序与在顺序一致性模型中的执行结果相同。
3. 未同步/未正确同步的多线程程序。JMM为它们提供了最小安全性保障：线程执行时读取到的值，要么是之前某个线程写入的值，要么是默认值（0、null、false）。

## 2. JMM 需要解决的问题

由于编译器、CPU 对指令进行了重排序，以及 CPU 缓存的存在，使得实现 JMM 的规范，需要采取一定的措施。

### 2.1 重排序

1. 编译器优化的重排序。编译器不改变单线程程序的语义前提下，重排语句顺序。
2. 指令级并行的重排序。CPU 的流水线技术。如果不存在数据依赖性，CPU 可以改变机器指令的执行顺序。

#### CPU 重排序类型

不同 CPU 允许的重排序类型如下。

![CPU重排序规则](/images/CPU重排序规则.png)

注意前 4 个规则中，操作之间没有数据依赖。编译器、CPU 不会改变**单线程**中存在**数据依赖**关系的两个操作的执行顺序。

### 2.2 CPU 缓存

由于 CPU 的多级缓存的存在，多个线程对同一个变量的读取值可能是不同的。

由于 CPU 缓存的存在，在 32 位机器上，`long` 和 `double` 的读取和写入不是原子的。

后文的线程**本地内存**指 CPU 的缓存等技术。

## 3. 解决办法

- 对于编译器的重排序，Java 编译器**重排序规则**，来禁止特定类型的重排序。
- 对于处理器的重排序、缓存不一致，编译器插入**内存屏障**指令，来禁止特定类型的 CPU 的重排序、同时刷新/无效化缓存。

### 内存屏障类型

![内存屏障类型](/images/%E5%86%85%E5%AD%98%E5%B1%8F%E9%9A%9C%E7%B1%BB%E5%9E%8B.png?lastModify=1582630839)

StoreLoad Barriers 同时具备其他三个屏障的效果，因此也称之为**全能屏障**，是目前大多数处理器所支持的；但是相对其他屏障，该屏障的开销相对昂贵。

## 4. volatile 的内存语义

### 4.1 特性

- 可见性。对 `volatile` 变量的读，总是能看到（任意线程）对这个 `volatile` 变量最后的写入。
- 原子性：对任意单个 `volatile` 变量的读/写具有原子性，但类似于 `volatile++` 这种复合操作不具有原子性。

### 4.2 内存语义

- 写一个 `volatile` 变量时，JMM 把该线程对应的本地内存中的共享变量值刷新到主内存。
- 读一个 `volatile` 变量时，JMM会把该线程对应的本地内存置为无效。

### 4.3 内存语义的实现

#### 虚拟机重排序限制

JMM 限制如下的指令重排序：

![volatile编译器重排序规则](/images/volatile编译器重排序规则.png)

- `volatile` 写操作，不能与它前面的指令重排序。
- `volatile` 读操作，不能与它后面的指令重排序。
- 当第一个操作是 `volatile` 写，第二个操作是 `volatile` 读时，不能重排序。

#### CPU 内存屏障

##### 保守的实现方式

编译器在生成指令序列时，插入相应的内存屏障。

- 在每个 `volatile` 写操作的前面插入一个 `StoreStore` 屏障。
  禁止前面普通写和 `volatile` 写重排序。
- 在每个 `volatile` 写操作的后面插入一个 `StoreLoad` 屏障。
  禁止这个`volatile` 写与后面可能存在的 `volatile` 读重排序。（因为读线程数目一般多于写线程数目，所以不是在`volatile` 读前面加入 `StoreLoad` 屏障）。
- 在每个 `volatile` 读操作的后面插入一个 `LoadLoad` 屏障。
  禁止后面的普通读操作与 `volatile` 读重排序。
- 在每个 `volatile` 读操作的后面插入一个 `LoadStore` 屏障。
  禁止后面的普通写与 `volatile` 读重排序。

##### 实际实现方式

以上保守实现方式，在不同的 CPU 上实现不同，因为 CPU 本身机会禁止一些重排序。

例如，X86 只会对写-读操作做重排序，JMM 只需在 `volatile` 写后面插入 `StoreLoad`。因此，X86 上，`volatile` 写开销比 `volatile` 读开销大很多。

## 5. 锁的内存语义

### 内存语义

- 当线程获取锁时，JMM 把该线程对应的本地内存置为无效。

- 当线程释放锁时，JMM 把该线程的本地内存的共享变量刷新到主内存中。

### 内存语义的实现

#### 监视器锁

在 `monitorenter`和`monitorexit` 两个指令的前后插入内存屏障。

#### Lock 类

`AQS` 内部使用 `volatile` 的 `state` 变量来表示锁的状态，在获取锁和释放锁的时候，要进行 `volatile` 的读取、写入、CAS。

其中 CAS 会在指令上加 `lock` 前缀，相当于同时具有volatile读和volatile写的内存语义。

方腾飞,魏鹏,程晓明 著. Java并发编程的艺术 (Java核心技术系列) (Chinese Edition) (Kindle位置1218). Kindle 版本. 。

intel的手册对lock前缀的说明如下。

1. 确保对内存的读改写操作原子执行。使用总线锁或缓存锁。
2. 禁止该指令，与之前和之后的读和写指令重排序。
3. 把写缓冲区中的所有数据刷新到内存中。

### 线程间通信的方法

1. A 线程写 `volatile` 变量，随后 B 线程读这个 `volatile` 变量。
2. A 线程写 `volatile` 变量，随后 B 线程用 CAS 更新这个 `volatile` 变量。
3. A 线程用 CAS 更新一个 `volatile` 变量，随后B线程用 CAS 更新这个 `volatile` 变量。
4. A 线程用 CAS 更新一个 `volatile` 变量，随后B线程读这个 `volatile` 变量。

## 6. final 域的内存语义

### 6.1 final 域的重排序规则

1. 在构造函数内对一个 `final` 域的写入，与随后把这个被构造的对象的引用 赋值给一个引用变量，这两个操作之间不能重排序。

2. 初次读一个包含 `final` 域的对象的引用，与随后初次读这个对象的 `final` 域，这两个操作之间不能重排序。

### 6.2 写 final 域的重排序规则

1. 对于编译器。JMM 禁止编译器把 `final` 域的写重排序到构造函数之外。
2. 对于 CPU。编译器会在 `final` 域的写之后、构造函数 return 之前，插入一个 StoreStore 屏障。这个屏障禁止处理器把 `final` 域的写重排序到构造函数之外。

### 6.3 读 final 域的重排序规则

对于CPU。编译器会在读 `final` 域操作的前面插入一个 LoadLoad 屏障。来实现禁止 初次读对象引用与初次读该对象的的 `final` 域。

这两个操作之间存在间接依赖关系。

- 编译器遵守间接依赖关系，因此编译器不会重排序这两个操作。
- 大多数处理器也会遵守间接依赖，只有少数 CPU 需要加 LoadLoad 屏障。

### 6.4 final 域为引用类型

对与编译器、CPU。在构造函数内对一个 `final` 引用对象的成员域的写入，与随后在构造函数外把这个被构造对象的引用赋值给一个引用变量，这两个操作之间不能重排序。

例如，A 与 C 不能重排序，B 与 C也不能重排序。

```java
Class Test {
  private final int[] nums;
  
  public Test() {
    nums = new int[2];  // A
    nums[0] = 1;				// B
  }
  
  public writeRef() {
    nums[1] = 2;				// C
  }
}
```

### 6.5 构造函数的溢出

即使有 `final` 域的内存语义，也不能保证构造函数是安全初始化的，必须保证在构造函数内部，不能让这个被构造对象的引用为其他线程所见。

例如，由于 A 和 B 可以重排，D 处可能在 `num = 1` 之前就进行了读取。

```java
Class Test {
  private static Test instance;
  
  private final int num;
  
  public Test() {
    num = 1;  							// A
    instance = this;				// B
  }
  
  public static write() {
    new Test()							// C
  }
  
  public static read() {
    if (instance != null) {
      System.out.println(instance.num);	// D
    }
  }
}
```

## 7. 单例模式延迟初始化与双重检查

### 双重检查

为了使单例模式延迟初始化线程安全，并且避免每次都检查锁，而使用了双重检查。

```java
class DoubelCheckedSingleton {
  private static DoubelCheckedSingleton instance;
  
  public static DoubelCheckedSingleton getInstance() {
    if (instance == null) {
      synchronized(DoubleCheckedSingleton.class) {
        instance = new DoubleCheckedSingleton(); 			// A
      }
    }
    return instance;
  }
}
```

### 双重检查的 bug

代码 A 处相当于

```java
memory = allocate();  	// 1. 分配对象的内存空间
initInstance(memory);		// 2. 初始化对象
instance = memory;			// 3. 把 instance 指向 memory 的内存地址
```

2 和 3 可能发生重排，这样线程可能看到一个还没有初始化的对象。

解决方法：

1. 不允许 2 和 3 重排序。
2. 允许 2 和 3 重排序，但不允许其他线程“看到”这个重排序。

### 基于 volatile 解决方案

为了使 2 和 3 不能重排序。把 `instance` 设为 `volatile` 即可。

```java
class DoubelCheckedSingleton {
  private static volatile DoubelCheckedSingleton instance;
  
  public static DoubelCheckedSingleton getInstance() {
    if (instance == null) {
      synchronized(DoubleCheckedSingleton.class) {
        instance = new DoubleCheckedSingleton(); 			// A
      }
    }
    return instance;
  }
}
```

### 基于类初始化的解决方案

```java
public class Singleton {
  private static class InstanceHolder {
    static Instance instance = new Singleton(); 	
  }
  public static Singleton getInstance() {
    return InstanceHolder.getInstance();
  }
}
```

在使用一个类的静态域时，这个类会被初始化，而初始化的时候会加锁。所以使得只有一个线程可以进行初始化，且其他线程无法看到整个过程。

## 8. 编译器和 CPU 对代码的优化处理

### 条件分支的猜测执行

控制依赖，会影响指令序列的并行度。

```java
if (flag) {
  int y = x * x;
}

// 可能会被优化为
int tmp = x * x;
if (flag) {
  int y = tmp;
}
```

