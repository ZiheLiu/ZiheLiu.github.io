---
title: LinkedHashMap 源码分析
date: 2020-03-04 14:48:52
tags:
	- java
categories:
	- java
	- basic
typora-root-url: ../../../
---

# 引用

- [搞懂 Java LinkedHashMap 源码](https://juejin.im/post/5ace2bde6fb9a028e25deca8)

# 与 HashMap 的关系

- `LinkedHashMap` 继承自 `HashMap`，复用了 `HashMap` 的所有结构、操作和机制。

- 使用双向链表，记录节点的访问顺序。

  - 在 `accessOrder` 为 false 时，只有插入元素时，会把访问的节点放在链表结尾。

  - 在 `accessOrder` 为 true 时，只要访问/更新元素，就会把访问的节点挪到链表结尾。


# 实现方式

## 扩展节点

```java
static class Entry<K,V> extends HashMap.Node<K,V> {
   Entry<K,V> before, after;
   Entry(int hash, K key, V value, Node<K,V> next) {
       super(hash, key, value, next);
   }
}
```

在节点中，添加 before 和 after，来实现双向链表。

## 创建节点

```java
// LinkedHashMap
Node<K,V> newNode(int hash, K key, V value, Node<K,V> e) {
    LinkedHashMap.Entry<K,V> p =
        new LinkedHashMap.Entry<K,V>(hash, key, value, e);
    linkNodeLast(p);
    return p;
}

// HashMap
Node<K,V> newNode(int hash, K key, V value, Node<K,V> next) {
  	return new Node<>(hash, key, value, next);
}
```

相比 `HashMap`，在创建节点后，会把节点加入到链表尾部。

## 双向链表操作

```java
void afterNodeRemoval(Node<K,V> e) { // unlink
    LinkedHashMap.Entry<K,V> p =
        (LinkedHashMap.Entry<K,V>)e, b = p.before, a = p.after;
    p.before = p.after = null;
    if (b == null)
        head = a;
    else
        b.after = a;
    if (a == null)
        tail = b;
    else
        a.before = b;
}

void afterNodeInsertion(boolean evict) { // possibly remove eldest
    LinkedHashMap.Entry<K,V> first;
    if (evict && (first = head) != null && removeEldestEntry(first)) {
        K key = first.key;
        removeNode(hash(key), key, null, false, true);
    }
}

void afterNodeAccess(Node<K,V> e) { // move node to last
    LinkedHashMap.Entry<K,V> last;
    if (accessOrder && (last = tail) != e) {
        LinkedHashMap.Entry<K,V> p =
            (LinkedHashMap.Entry<K,V>)e, b = p.before, a = p.after;
        p.after = null;
        if (b == null)
            head = a;
        else
            b.after = a;
        if (a != null)
            a.before = b;
        else
            last = b;
        if (last == null)
            head = p;
        else {
            p.before = last;
            last.after = p;
        }
        tail = p;
        ++modCount;
    }
}
```

- 在调用 `remove()` 时，使用 `afterNodeRemoval()` 把节点从链表删除。
- 在调用 `get()`、`put` 更新已经存在的节点，使用 `afterNodeAccess()` 把节点挪到链表尾部。
- 在插入新的节点后，调用 `afterNodeInsertion()`，如果 `removeEldestEntry()` 为 true，则删除头结点（最旧的节点）。
  - LinkedHashMap 中 `removeEldestEntry()` 会返回 false，这个功能是给子类用的。