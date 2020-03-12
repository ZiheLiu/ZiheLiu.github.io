---
title: Recycler 对象池
date: 2020-03-09 15:34:00
categories:
	- netty
tags:
	- netty
typora-root-url: ../../
---

# 引用

- [AtomicInteger lazySet vs. set](https://stackoverflow.com/questions/1468007/atomicinteger-lazyset-vs-set)
- [why use lazySet instead set in WeakOrderQueue#add](https://github.com/netty/netty/issues/8215)
- [冷知识：netty的Recycler对象池](https://blog.csdn.net/alex_xfboy/article/details/90384332)

# 介绍

Netty 场景下的轻量级对象池 Recycler。

# 数据结构

## stack

每个线程使用一个 `stack` 存储对象池。避免线程同步的问题。

```java
private final FastThreadLocal<Stack<T>> threadLocal
```

## WeakOrderQueue

在回收对象时，要释放对象的线程可能不是对象的 `stack` 线程。

所以对于**每个** `stack`，**每个**线程存储了一个对象的队列，供之后把对象放入到 `stack` 中。

```java
private static final FastThreadLocal<Map<Stack<?>, WeakOrderQueue>> DELAYED_RECYCLED =
        new FastThreadLocal<Map<Stack<?>, WeakOrderQueue>>() {
    @Override
    protected Map<Stack<?>, WeakOrderQueue> initialValue() {
        return new WeakHashMap<Stack<?>, WeakOrderQueue>();
    }
};
```

### stack 中记录 queue

且 `stack` 中，以**链表**的形式，存储了在多个线程中创建的它的 queue。

实现方式是，在创建 queue 时，把它加入到 `stack` 的头结点中。

```java
static WeakOrderQueue newQueue(Stack<?> stack, Thread thread) {
    final WeakOrderQueue queue = new WeakOrderQueue(stack, thread);
    stack.setHead(queue);

    final Head head = queue.head;
    ObjectCleaner.register(queue, head);

    return queue;
}
```

### 节点内容

节点 `Link` 继承自 `AtomicInteger`，内置了一个对象数组（默认16），在元素满了后，会创建下一个节点。

继承自 `AtomicInteger` 本身的值作为 `writerIndex`。通过 `AtomicInteger` 保证了**写索引**的可见性。

内置一个 `int` 的 `readIndex`，由于只有 `stack` 线程会去读，**不用保证线程安全**。

```java
static final class Link extends AtomicInteger {
    private final DefaultHandle<?>[] elements = new DefaultHandle[LINK_CAPACITY];

    private int readIndex;
    Link next;
}
```

# 取对象 allocate()

- 从 `stack` 中取一个对象。
- 如果取不到，去 queue 的链表中扫描，找到可使用的对象。
- 如果找不到，创建一个对象。

# 释放对象 deallocate()

- 当调用的线程是 `stack` 的拥有者线程，把对象放入到 `stack` 里。
  - 每 `RATIO` 个才会放进去一个，默认是 8。
  - 如果对象池大小超过阈值，不放进去。
- 否则，是其他线程在操作。
  - 把元素放到队列里。
  - 在放入时，使用了 `lazySet()` 来更新 `writerIndex`。

## lazySet()

### lazySet() 功能

- `lazySet()` 只会在写之前插入 `StoreStore` 屏障，保证与前面写操作的可见性。
- 不会像 `voltatile` 写一样，在写之后插入 `StoreLoad` 屏障，保证与后面操作的可见性。 `StoreLoad` 屏障代价比较大。

### 使用的作用

- 保证在看到 `tail == writerIndex + 1` 之前，看到 `handle.stack` 被设为 null 了。

- 这样可能会有一段时间，`stack` 线程看不到这个新释放的对象，是可以容忍的。
- 但是可以不用插入 `StoreLoad` 屏障。

```java
void add(DefaultHandle<?> handle) {
    handle.lastRecycledId = id;

    Link tail = this.tail;
    int writeIndex;
    if ((writeIndex = tail.get()) == LINK_CAPACITY) {
        if (!head.reserveSpace(LINK_CAPACITY)) {
            // Drop it.
            return;
        }
        // We allocate a Link so reserve the space
        this.tail = tail = tail.next = new Link();

        writeIndex = tail.get();
    }
    tail.elements[writeIndex] = handle;
    handle.stack = null;

    tail.lazySet(writeIndex + 1);
}
```