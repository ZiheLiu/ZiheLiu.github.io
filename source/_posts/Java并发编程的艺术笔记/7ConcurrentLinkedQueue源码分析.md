---
title: 7. ConcuurentLinkedQueue 源码分析
date: 2020-03-04 14:48:07
tags:
	- java
	- concurrency
categories:
	- java	
	- concurrency
typora-root-url: ../../
---

# 7. ConcuurentLinkedQueue 源码分析

## 0. 引用

- [[ConcurrentLinkedQueue 源码解读](https://www.cnblogs.com/jmcui/p/11433016.html)](https://www.cnblogs.com/jmcui/p/11433016.html)
- [JUC源码分析-集合篇（三）ConcurrentLinkedQueue](https://www.cnblogs.com/binarylei/p/10923374.html)

## 1. 准备

### Node

```java
private static class Node<E> {
    volatile E item;
    volatile Node<E> next;
}
private transient volatile Node<E> head;
private transient volatile Node<E> tail;
```

`Node` 中的 `item` 和 `next` 使用 `volatile` 来保证可见性、避免重排序。

### 首尾节点

`head` 和 `tail` 并不是每次操作都会进行更新，而是在操作时发现首节点/尾节点发生了变化（与`head`和`tail` 不同时）才会更新。减少了 CAS 竞争的概率，提高了并发。

因此，`head` 和 `tail` 中存储的可能并不是当前的首节点和尾节点。

### CAS 更新

```java
private static class Node<E> {
    boolean casItem(E cmp, E val) {
      return UNSAFE.compareAndSwapObject(this, itemOffset, cmp, val);
    }

    void lazySetNext(Node<E> val) {
      UNSAFE.putOrderedObject(this, nextOffset, val);
    }

    boolean casNext(Node<E> cmp, Node<E> val) {
      return UNSAFE.compareAndSwapObject(this, nextOffset, cmp, val);
    }
}
private boolean casTail(Node<E> cmp, Node<E> val) {
    return UNSAFE.compareAndSwapObject(this, tailOffset, cmp, val);
}

private boolean casHead(Node<E> cmp, Node<E> val) {
    return UNSAFE.compareAndSwapObject(this, headOffset, cmp, val);
}
```

直接使用 `unsafe` 的底层的 CAS 方法，不使用 `AtomicXXX` 类，避免性能损耗。

- `AtomicXXX` 和 这里用的方法是一样的，都是 `UNSAFE.compareAndSwapObject` + `offset`。
- 避免再套一层对象。

## 2. 构造函数

```java
public ConcurrentLinkedQueue() {
    head = tail = new Node<E>(null);
}
```

首尾节点都设为空的虚拟 Node 节点。

## 3. 入队列 offer()

```java
public boolean offer(E e) {
  	// 节点不能为 null
    checkNotNull(e);
    final Node<E> newNode = new Node<E>(e);

  	// t 是 tail 的快照
  	// 等会要把节点插入到 p 后边
    for (Node<E> t = tail, p = t;;) {
        Node<E> q = p.next;
      	// 1. 说明 p 是最后一个节点。
        if (q == null) {
            // 1.1 CAS 把节点插入到 p 后边
            if (p.casNext(null, newNode)) {
                // 1.1.1 p 是最后一个节点，却不等于 tail。说明 tail 不是最新的尾节点了。
              	// 		所以更新 tail 为最新的尾节点
                if (p != t) // hop two nodes at a time
                    casTail(t, newNode);  // 失败了也没有关系，可能是之后加入的节点更新了 tail
                return true;
            }
            // Lost CAS race to another thread; re-read next
        }
      	// 2. p 的 next 指向了自己（在 poll() 中做的），说明 p 节点被删除了。
      	// 		因为 p 是最后一个节点，所以重新读取 tail 的快照。
        else if (p == q)
            // 2.1 重新取 tail 的快照，从新的 tail 开始遍历。
          	// 2.2 如果没有变化，说明 tail 也被删除了，只能从 head 开始遍历。
            p = (t != (t = tail)) ? t : head;
        else
            // 3.1 如果 p 与 t 相等，说明 p 不是最后一个节点（因为 p 有后继节点），
          	//		继续向 p 的后继节点遍历。
          	// 3.2 如果 p 与 t 不相等，说明尾节点发生变化了，
          	//		重新读取 tail 的快照，从新的 tail 开始遍历。
            p = (p != t && t != (t = tail)) ? t : q;
    }
}
```

-  从 tail 开始遍历查找到最新的尾节点 p ，找到后 CAS 把新加的节点插入 p 后边。

- 只有查找中，发现 tail 不是最新的尾节点，才会 CAS 更新 tail。

## 4. 出队列 poll()

```java
public E poll() {
    restartFromHead:
    for (;;) {
      	// h 是 head 的快照。
      	// p 是等会要删除的节点，从 head 开始遍历。
        for (Node<E> h = head, p = h, q;;) {
            E item = p.item;

          	// 1 把 p.item CAS 设为 null，如果成功，说明节点被删除。
          	//		注意，还需后续操作：把 p.next 指向自己、更新 head。
            if (item != null && p.casItem(item, null)) {
                // 如果 p 不等于 h，说明 p 经过遍历才删除了节点，即 head 现在不是最新的首节点。
              	// 		所以，更新首节点到 head、把 p.next 指向自己。
                if (p != h)
                  	// a. 如果 p（被删除了） 有下一个节点，则设首节点为 p 的下一个节点。
                  	// b. 否则，只能把首节点设为 p 了，此时首节点是一个虚拟节点。
                    updateHead(h, ((q = p.next) != null) ? q : p);
                return item;
            }
          	// 2. 和上边类似。如果 p 没有下一个节点了，只能把 p 设为首节点了。
            else if ((q = p.next) == null) {
                updateHead(h, p);
                return null;
            }
          	// 3. 如果 p.next 指向自己，说明 p 被删除了。需要从新的 head 开始。
            else if (p == q)
                continue restartFromHead;
          	// 4. 否则，遍历下一个节点。
            else
                p = q;
        }
    }
}
final void updateHead(Node<E> h, Node<E> p) {
    if (h != p && casHead(h, p))
      h.lazySetNext(h); // 不用 CAS
}
```

- 从 head 开始遍历查找最新的首节点 p，CAS 更新 p.item 为 null。
- 只有查找中，发现 head 不是最新的首节点，才会 CAS 更新 head、把旧的 head 指向自己。

## 5. 删除节点 remove()

```java
public boolean remove(Object o) {
    if (o != null) {
        Node<E> next, pred = null;
      	// 从首节点开始遍历
        for (Node<E> p = first(); p != null; pred = p, p = next) {
            boolean removed = false;
            E item = p.item;
            if (item != null) {
              	// 没有找到
                if (!o.equals(item)) {
                    next = succ(p);
                    continue;
                }
              	// 找到了，尝试 CAS 把 p.item 设为 null。抢夺删除权。
                removed = p.casItem(item, null);
            }

            next = succ(p);
          	// 把 p 前后节点连起来，注意可以和得到删除权的线程不同。
            if (pred != null && next != null) // unlink
                pred.casNext(p, next);
            if (removed)
                return true;
        }
    }
    return false;
}

Node<E> first() {
    restartFromHead:
    for (;;) {
      for (Node<E> h = head, p = h, q;;) {
        boolean hasItem = (p.item != null);
        // 找到了 p.item != null 的节点。
        // 或遍历到尾部了。
        if (hasItem || (q = p.next) == null) {
          updateHead(h, p);
          return hasItem ? p : null;
        }
        // 节点指向自己，从头开始遍历。
        else if (p == q)
          continue restartFromHead;
        else
          p = q;
      }
    }
}

final Node<E> succ(Node<E> p) {
    Node<E> next = p.next;
  	// 如果 p.next 指向自己，说明 p 已经被从 poll() 中删除了。
    return (p == next) ? head : next;
}
```

- 遍历链表，找到要删除的节点 p 后，使用 CAS 成功把 p.item 设为 null的，获得本次返回 true 的权利。其他线程要继续遍历查找。

- 找到节点后，要把 p 前后节点 CAS 连起来，可以和获得删除权的节点不同。
- `first()` 找首节点的函数，也会更新 head 为最新的首节点。

## 6. 获取节点数目 size()

```java
public int size() {
    int count = 0;
    for (Node<E> p = first(); p != null; p = succ(p))
        if (p.item != null)
            // Collection.size() spec says to max out
            if (++count == Integer.MAX_VALUE)
                break;
    return count;
}
```

遍历整个链表，找到 item 不为 null 的节点。

`size()` 是 O(n) 的时间复杂度。

## 7. 元素是否为空 isEmpty()

```java
public boolean isEmpty() {
    return first() == null;
}
```