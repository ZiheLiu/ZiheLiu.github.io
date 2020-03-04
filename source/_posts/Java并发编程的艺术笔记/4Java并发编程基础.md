---
title: 4. Java 并发编程基础
date: 2020-03-04 14:48:04
tags:
	- java
	- concurrency
categories:
	- java	
	- concurrency
typora-root-url: ../../
---

# 4. Java 并发编程基础

Java 的线程对应操作系统级别的线程。

执行 `main()` 方法的是主线程。

## 1. 线程的状态

Java 线程的 6 个状态：

![Java线程的状态](/images/Java线程的状态.png)

线程状态的迁移：

![Java线程状态的迁移](/images/Java线程状态的迁移.png)

## 2. Daemon 线程

当 JVM 中不存在非 Daemon 线程时，JVM 将会推出。

使用 `thread.setDaemon(true)` 来设置，只能在线程启动前设置。

## 3. 线程初始化

线程 A 的父线程（创建线程 A 的线程）来初始化线程 A。

- 子线程继承了 父线程的 `daemon`、优先级、contextClassLoader、InheritiableThreadLocal
- 分配 `threadID` (从 0 开始递增)。

## 4. 启动线程

调用 `thread.start()` 方法来启动线程，它会调用 `thread.run()` 方法。

`Thread` 类 默认的 `start` 方法是调用 `Runnable` 类型的 `target` 的 `target.run()` 方法。

因此，可以在构造线程时，传入一个 `Runnable` 对象。

## 5. 线程的中断

### 语法

注意，这里的中断不是操作系统的中断，而是线程的一个标志位。

常用方法：

- `threadA.interrupt()` 其他线程调用来中断 A 线程。
- `threadA.isInterrupted()` 返回 A 线程是否被中断了。如果线程状态是 `TERMINATED`，即使被中断过，也会返回 false。
- `Thread.interrupted()` 静态方法，返回当前线程是否被中断了，并且清除标志位。

许多方法声明了 `throws InterruptedException`，在执行这些方法时，如果线程被中断，则会抛出 `InterruptedException`、清除标志位。

### 终止线程

#### suspend()、resume()、stop()

`suspend()`、`resume()`、`stop()` 方法可以暂停、回复、终止线程。

但是他们不给线程资源释放的时间、没办法释放锁，只要调用了，就会进行操作，所以现在是**过期**的方法了。

#### 可以使用中断来安全的终止线程

```java
public class MyRunnable implements Runnable {
  @override
  public void run() {
    while (!Thread.interrupted()) {
      // do somthing ...
    }
    // 释放资源
  }
}

Thread thread = new Thread(new MyRunnable());
thread.start();
thread.interrupt(); // 终止线程
```

## 6. 等待/通知机制

### 原理

`Object` 类上定义了 `wait()` 和 `notify()` 方法。

- 持有锁的线程，调用 `wait()` ，会释放锁，阻塞在 `wait()` 方法上。这个线程被移动到这个 `Object` 的监视器的等待队列上。
- 持有锁的线程 A，调用 `notify()`，
  - 会把一个线程 B 从监视器的等待队列移到同步队列，从而 B 开始竞争锁。
  - 在线程 A 释放锁后（注意，不是调用了 `notify()` 就释放锁），B 获得锁后，从 `wait()` 方法返回。
- `notifyAll()`，把监视器的等待队列上的所有线程都移动到同步队列上。

### 经典范式

```java
// 等待方（消费者）
synchronized(obj) {
  while (条件不满足) {
    obj.wait();
  }
  条件满足了，进行逻辑处理
}

// 通知方（生产者）
synchronized(obj) {
  进行逻辑处理，改变条件
  obj.notifyAll();
}
```

## 7. thread.join() 方法

#### 语法

线程 A 执行了 `threadB.join()` 方法，则线程 A 需要等待线程 B 终止之后才能从 `threadB.join()` 方法返回。

#### 原理

- 调用 `threadB.join()` 方法，其实是线程 A 调用了 `threadB.wait(0)` 方法。注意需要先 `synchronized` 得到 `threadB` 的监视器锁。
- 在线程 B 终止时，会调用 `threadB.notifyAll()` 方法。

```java
public final synchronized void join(long millis) {
  while (isAlive()) {
    wait(0);
  }
}
```

## 8. ThreadLocal

### 8.0 引用

- [ThreadLocal源码深度剖析](https://juejin.im/post/5a5efb1b518825732b19dca4)
- [源码|ThreadLocal的实现原理](https://juejin.im/post/59db31c16fb9a00a4843dc36)
- [[java 静态变量生命周期（类生命周期）](https://www.cnblogs.com/hf-cherish/p/4970267.html)](https://www.cnblogs.com/hf-cherish/p/4970267.html)

### 8.1 语法

`ThreadLocal` 类提供了线程内部的局部变量。

```java
ThreadLocal<String> strLocal = ThreadLocal.withInitial(() -> new String("test"));
strLocal.set(new String("hello"));
System.out.println(strLocal.get());
```

### 8.2 原理

![threadlocal](/images/threadlocal.png)

#### 数据结构

ThreadLocal 的数据结构如上图所示。

- 每个 Thread 存储了一个 `ThreadLocalMap`，是一个内部实现的哈希表。

- `ThreadLocalMap` 内部存储了一个 `Entry` 数组，初始大小是 16。
- `Entry` 是 `WeakReference<ThreadLocal<?>>` 的子类，存储了 key 的弱引用和 value。

#### 弱引用的 key

通过弱引用，实现了**线程封闭**的线程局部变量。

- 当创建 ThreadLocal 变量的对象 GC 后，如果不使用弱引用，因为 entry 还引用着这个对象作为 key，没有办法 GC 这个 ThreadLocal 对象。
- 而使用弱引用，如果只有 entry 引用了 key，则key会被回收，调用 key.get() 会得到 null。

但是注意，只是 entry 的 key 被 GC 了，value 还没有GC。因此，`ThreadLocal` 的 `get()`、`remove()`、`set()` 方法中会检查一部分的 entry，看 key 是否为null，如果是的话把 value 设为 null，使得 value 可以被 GC。

#### hash 算法

Entry 数组初始大小是 16，当遇到碰撞时，使用寻址法，找到下一个可用的空位置。

当负载达到 `threshold`（2/3 * 数组的长度）后，进行 rehash。

- 首先遍历所有 Entry，清除 key 为 null 的元素。
- 如果 `size >= threshold - threshold / 4`，把数组长度加倍，重新放元素。

### 8.3 内存泄漏

1. 如果一个线程的 ThreadLocal 变量定义后，再也没有用过，则没有机会调用 `get`、`set` 方法，`value` 永远也没有办法被回收。

2. 此外，如果是使用 `static` 声明的 ThreadLocal 对象，一般生命周期为整个应用程序，这个对 key 的引用一直存在，则没有办法回收 key。

> `static` 的生命周期：
>
> - 加载：JVM 在加载类的过程中为静态变量分配内存。
> - 销毁：类被卸载时，静态变量被销毁，并释放内存空间。
>
> 类的生命周期：
>
> - 该类所有的实例都已经被回收，也就是 Java 堆中不存在该类的任何实例。
> - 加载该类的 ClassLoader 已经被回收。
> - 该类对应的 java.lang.Class 对象没有任何地方被引用，无法在任何地方通过反射访问该类的方法。

所以，在不再使用 ThreadLocal 对象时，主动调用 `remove()` 方法来删除它。

