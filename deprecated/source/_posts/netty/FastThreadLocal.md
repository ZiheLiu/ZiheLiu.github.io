---
title: FastThreadLocal
date: 2020-03-05 23:31:00
categories:
	- netty
tags:
	- netty
---

# 引用

- [Netty 高性能之道 FastThreadLocal 源码分析（快且安全）](https://www.jianshu.com/p/3fc2fbac4bb7)

# 数据结构

## threadLocalMap

每个线程内部有一个 `threadLocalMap` 。

其内部使用 `Object[] indexedVariables` 存储了每一个本地线程变量。

- 如果这个线程时 Netty 的 `FastThreadLocalThread` 线程，它的内部存有 `threadLocalMap` 域。
- 如果是原生的线程，使用 原生的 `ThreadLocal` 存储每个原生线程的 `threadLocalMap`。

## 自增 index

每创建一个 `FastThreadLocal`，会使用一个全局的 `AtomicInteger` 自增作为 `index`。

稍后 value 会存储在 `threadLocalMap.indexedVariables[index]` 处。

- 注意，多个线程的 `index` 是自增的，会浪费空间。

```java
// class FastThreadLocal
public FastThreadLocal() {
    index = InternalThreadLocalMap.nextVariableIndex();
}

// class InternalThreadLocalMap
static final AtomicInteger nextIndex = new AtomicInteger();
public static int nextVariableIndex() {
    int index = nextIndex.getAndIncrement();
    if (index < 0) {
        nextIndex.decrementAndGet();
        throw new IllegalStateException("too many thread-local indexed variables");
    }
    return index;
}
```

## toRemoveSet

一个线程占用了一处 `threadLocalMap.indexedVariables` 来存储 `toRemoveSet`。

`toRemoveSet` 存储了需要删除的 `FastThreadLocal` 对象。

# set()

调用 ``fastThreadLocal.set(val)` 时，如果是新增而不是更新值：

- 把 `fastThreadLocal` 对象加入`toRemoveSet`。
- 把 `fastThreadLocal` 对象同时注册一个 `ObjectCleaner`，在这个线程被 GC 时，会调用 `fastThreadLocal.remove()`。

# remove()

在调用 `fastThreadLocal.remove()` 时，

- 把 `indexedVariables` 对应 `index` 设为 UNSET。
- 把该对象从 `toRemoveSet` 删除。

# removeAll()

调用 `FastThreadLocal.removeAll()` （类的静态方法）

- 从本线程的 `indexedVariables` 中取出`toRemoveSet`。
- 遍历 `toRemoveSet` 每一个 `fastThreadLocal` 对象，调用 `remove()`。

# ObjectCleaner

## FastThreadLocal#set()

在 `FastThreadLocal#set(val)` 时，如果是新增的，会注册 GC 时的行为。

如下，把当前线程、要执行的函数传给 `ObjectCleaner.register()`。

```java
ObjectCleaner.register(current, new Runnable() {
    @Override
    public void run() {
        remove(threadLocalMap);
    }
});
```
## ObjectCleaner.register()

把线程封装为弱引用，那弱引用与 `REFERENCE_QUEUE` 引用队列关联起来。

把这个弱引用放入 `LIVE_SET`。

启动 `cleanupThread` 来运行 `CLEANER_TASK`。

```java
private static final class AutomaticCleanerReference extends WeakReference<Object> {
    private final Runnable cleanupTask;
		// 弱引用 REFERENCE_QUEUE 放入队列。
    AutomaticCleanerReference(Object referent, Runnable cleanupTask) {
        super(referent, REFERENCE_QUEUE);
        this.cleanupTask = cleanupTask;
    }

    void cleanup() {
        cleanupTask.run();
    }

    @Override
    public Thread get() {
        return null;
    }

    @Override
    public void clear() {
        LIVE_SET.remove(this);
        super.clear();
    }
}

public static void register(Object object, Runnable cleanupTask) {
  	// 弱引用。
  	AutomaticCleanerReference reference = new AutomaticCleanerReference(object,
            ObjectUtil.checkNotNull(cleanupTask, "cleanupTask"));
    
    LIVE_SET.add(reference);

    // Check if there is already a cleaner running.
    if (CLEANER_RUNNING.compareAndSet(false, true)) {
      	// CLEANER_TASK 要执行的。
        final Thread cleanupThread = new FastThreadLocalThread(CLEANER_TASK);
        cleanupThread.setPriority(Thread.MIN_PRIORITY);
        AccessController.doPrivileged(new PrivilegedAction<Void>() {
            @Override
            public Void run() {
                cleanupThread.setContextClassLoader(null);
                return null;
            }
        });
        cleanupThread.setName(CLEANER_THREAD_NAME);

        cleanupThread.setDaemon(true);
        cleanupThread.start();
    }
}
```

##  CLEANER_TASK.run()

在 `LIVE_SET` 非空时，阻塞出队 `REFERENCE_QUEUE`。

- JVM 保证对象在被 GC 时，会加入到和它关联的引用队列中。
- 见  {% post_link java/jvm/2Java垃圾回收 垃圾回收四种引用类别  %}

出队成功后，调用 `reference.cleanup()`，即注册的方法。

```java
private static final Runnable CLEANER_TASK = new Runnable() {
    @Override
    public void run() {
        boolean interrupted = false;
        for (;;) {
            while (!LIVE_SET.isEmpty()) {
                final AutomaticCleanerReference reference;
                try {
                    reference = (AutomaticCleanerReference) REFERENCE_QUEUE.remove(REFERENCE_QUEUE_POLL_TIMEOUT_MS);
                } catch (InterruptedException ex) {
                    interrupted = true;
                    continue;
                }
                if (reference != null) {
                    try {
                        reference.cleanup();
                    } catch (Throwable ignored) {
                    }
                    LIVE_SET.remove(reference);
                }
            }
            CLEANER_RUNNING.set(false);

            if (LIVE_SET.isEmpty() || !CLEANER_RUNNING.compareAndSet(false, true)) {
                break;
            }
        }
        if (interrupted) {
            Thread.currentThread().interrupt();
        }
    }
};
```

# 优缺点

## 优点

- 时间效率很高。
  - 没有用 hash、以及线性寻址法。
  - `ThreadLocal` 还需要在各类操作中花费 log(N) 时间检查被回收的对象。
- 内存安全的，在线程被 GC 时，必然会被回收掉。
  - 而 `ThreadLocal` 有内存泄漏的风险。见 {% post_link java/Java并发编程的艺术笔记/4Java并发编程基础 ThreadLocal %}

## 缺点

- 空间效率低。
  - 所有线程共有自增的 `index`，每个线程的 `threadLocalMap` 有很多空洞。
  - `remove()` 调用后，虽然把位置设为了 `UNSET`，但是也不能再使用了。每次 `set()` 都是递增的 index。