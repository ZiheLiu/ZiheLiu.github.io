---
title: Netty 堆外内存
date: 2020-03-11 13:48:00
categories:
	- netty
tags:
	- netty
typora-root-url: ../../
---

# 引用

- [Netty堆外内存泄漏排查，这一篇全讲清楚了](https://segmentfault.com/a/1190000021469481)

- [Java堆外内存之三：堆外内存回收方法](https://www.cnblogs.com/duanxz/p/6089485.html)

# Java.nio 堆外内存

## 申请内存

`DirectByteBuffer` 通过 `unsafe.allocateMemory(size)` 申请堆外内存。

## 释放内存

### 手动释放

调用 `DirectByteBuffer#cleaner.clean()` 可以手动释放掉堆外的内存。

实际上通过调用 `unsafe.freeMemory(address)` 来释放的。

```java
public void run() {
    if (address == 0) {
        // Paranoia
        return;
    }
    unsafe.freeMemory(address);
    address = 0;
    Bits.unreserveMemory(size, capacity);
}
```

### 自动释放

由于 `DirectByteBuffer` 只是申请的堆外内存在堆内的包装对象，在 `DirectByteBuffer` 被垃圾回收时，内部的堆外内存并不会主动回收。

为了解决这个问题，而有了 `cleaner`。`DirectByteBuffer#cleaner` 继承自 `PhantomReference`。Cleaner 的 `refernence` 是 `DirectByteBuffer`，在 `DirectByteBuffer` 被垃圾回收时，reference-handler 线程会调用 `cleaner.clean()` 方法完成堆外内存的释放。

```java
DirectByteBuffer(int cap) {                   // package-private
    super(-1, 0, cap, cap);
    // 省略 ...
    cleaner = Cleaner.create(this, new Deallocator(base, size, cap));
}
```

但是如果 `DirectByteBuffer` 进入老年代，一般只有在 Full GC 的时候才会被垃圾回收。这就导致堆外内存很难被自动释放掉。

## 构造函数

在使用构造函数 `DirectByteBuffer(int cap)` 时，如果无法分配内存，会调用 `System.gc() `触发 Full GC后，再尝试分配内存，还是不能分配才会抛出 OOM。

注意，如果使用了 JVM 的 `-XX:+DisableExplicitGC` 则 `System.gc()` 将没有作用。

而其他的构造函数，例如 `DirectByteBuffer(long addr, int cap)` 只会存储地址、容量等信息，不会存储 `cleaner`、进行 `System.gc()` 等操作。

# Netty noCleaner 策略

在 Netty 4.1 后，引入了 noCleaner 策略。

使用 noCleaner 策略分配的 `DirectByteBuffer` 与不使用 noCleaner 策略的区别：

- 构造器不同
  - 不使用策略。使用 `DirectByteBuffer(int cap)` 构造器。
  - 使用策略。通过反射使用 `DirectByteBuffer(long addr, int cap)` 构造器。
- 释放内存不同。
  - 不使用策略。使用 `DirectByteBuffer#cleaner.clean()` 方法。
  - 使用策略。使用 `Unsafe.freeMemory(address)`。

Netty在启动时需要判断检查当前环境、环境配置参数是否允许noCleaner策略(具体逻辑位于PlatformDependent的static代码块)。例如运行在Android下时，是没有Unsafe类的，不允许使用noCleaner策略，如果不允许，则使用hasCleaner策略。

## 堆外内存的 JVM 参数

- `-XX:MaxDirectMemorySize` 在 `DirectByteBuffer(int cap)` 构造器中检查是否分配的内存超过了这个值。所以用 `Unsafe.allocateMemory(size)` 分配的内存不受它的控制。
- `-Dio.netty.maxDirectMemory`。用于限制 **noCleaner 策略下 Netty 的 DirectByteBuffer** 分配的最大堆外内存的大小，如果该值为 0，则使用 hasCleaner 策略

# Netty 释放内存方法

Netty 使用引用计数来释放内存。

- 使用一个 `AtomicInteger` 来记录引用的次数，在 `alloc()` 获取到 buffer 时，引用计数为 1。
- 调用`buffer.release()` 方法使引用计数减 1。
- 调用 `buffer.retain()` 方法使引用计数加 1。

Netty 在有些地方自动调用了 `buffer.release()` 方法。

- 调用 `ctx.write(buffer)` 在数据成功写入到 socket 之后，会自动调用一次 `buffer.release()`。
- `SimpleChannelInboundHandler#channelRead()` 最后会自动调用一次 `buffer.release()`。

## hasCleaner 策略为什么也要引用计数来释放内存

- 不使用引用计数，一般要 Full GC 才能释放内存。使用引用计数，再引用计数为 0 时马上可以释放内存。
- 在release() 掉 buffer 后，Netty 还有自己的处理逻辑，例如放回对象池。

# Netty 内存泄漏检测

检测 `ByteBuf` 对象被释放掉了，但是其对应的堆外内存没有释放的情况。

不能检测 `ByteBuf` 对象还没有释放的情况。

## 检测原理

- 使用 `DefaultResourceLeak` 来封装 buffer，
  - 它是 `WeakReference` 的子类。要检测的 buffer 是弱引用指向的对象。
- 使用 `ResourceLeakDetector.allLeaks` 记录了内存泄漏对象。

在需要检测内存泄漏时，检查 `DefaultResourceLeak` 对应的 referenceQueue 中的对象（即被 GC 掉的对象），如果它仍然在 `allLeaks` 里，说明内存泄漏了。

## 添加到 allLeaks 的时机

在创建一个 buffer 时，会调用 `ResourceLeakDetector#track(buffer)` 把 buffer 添加到 `allLeaks` 里。

```java
// ResourceLeakDetector#track0
private DefaultResourceLeak track0(T obj) {
    Level level = ResourceLeakDetector.level;
    if (level == Level.DISABLED) {
        return null;
    }

    if (level.ordinal() < Level.PARANOID.ordinal()) {
        if ((PlatformDependent.threadLocalRandom().nextInt(samplingInterval)) == 0) {
            reportLeak();
            return new DefaultResourceLeak(obj, refQueue, allLeaks);
        }
        return null;
    }
    reportLeak();
    return new DefaultResourceLeak(obj, refQueue, allLeaks);
}

// class DefaultResourceLeak
DefaultResourceLeak(
        Object referent,
        ReferenceQueue<Object> refQueue,
        Set<DefaultResourceLeak<?>> allLeaks) {
    super(referent, refQueue);

    assert referent != null;

    trackedHash = System.identityHashCode(referent);
    // 添加到 allLeaks 中。
    allLeaks.add(this);
    headUpdater.set(this, new Record(Record.BOTTOM));
    this.allLeaks = allLeaks;
}
```

## 从 allLeaks 删除的时机

在 `buffer.release()` 中发现引用计数变为 0 了，则调用 `DefaultResourceLeak#close()` 从 allLeaks 中移除。

```java
public boolean close() {
    if (allLeaks.remove(this)) {
        // Call clear so the reference is not even enqueued.
        clear();
        headUpdater.set(this, null);
        return true;
    }
    return false;
}
```

## 检查的时机

从 `track0(obj)` 函数中可以看到，是在创建 buffer 的时候，在 `track0(obj)` 中顺便检测内存泄漏的。

并且检测的频率根据设置的级别而不同。