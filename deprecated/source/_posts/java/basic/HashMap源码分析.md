---
title: HashMap源码分析
date: 2020-03-04 14:48:52
tags:
	- java
categories:
	- java	
	- basic
typora-root-url: ../../../
---

# 引用

- [深入理解HashMap: 关键源码逐行分析之hash算法](https://segmentfault.com/a/1190000015798586)

# 组成

## 节点

```java
static class Node<K,V> implements Map.Entry<K,V> {
    final int hash;
    final K key;
    V value;
    Node<K,V> next;
}
```

节点 `Node` 存储：

- 基本：hash 值、key值、value值。
- 多个值存在一个位置：链表 `next`。

## table

```java
Node<K,V>[] table;
```

节点存储在 `Node` 数组中。

## 节点的 key 如何映射到 table 的下标

```java
static final int hash(Object key) {
    int h;
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
int index = hash(key) & (table.length() - 1);
```

1. 使用 `hash()` 函数把 key 的hashCode 再进行hash。
   - 将 hashCode 的高16位和低16位进行异或，充分利用高半位和低半位的信息, 对低位进行了扰动。使该hashCode 映射成数组下标时可以更均匀。
2. table.length() 为 2 的 m 次幂，把第一步的 hash 值取低 m 位，来作为数组的下标。
3. key 可以是 null，这时 hash 值是 0。

## 对 key 的要求

- 哈希。用 `hashCode()` 方法调用 `HashMap#hash()` 得到 hash 值。
- 相等。用 `equals()` 比较 hash 值相同的节点。
- 比较。在红黑树时，会用到比较。

### 红黑树比较节点流程

在插入/更新元素到 map 时，需要找到红黑树中对应的节点。那么如何比较节点呢？

- 使用 `HashMap#hash(key.hashCode())` 比较 key。
  - 如果不同，找到了节点的大小关系。
  - 如果相同，使用 `equals()` 比较 key。
    - 如果相等，找到了需要更新的节点。
    - 如果不同。看 key 是否实现了 `Comparable`，
      - 如果实现了、且两个节点的 key 不等，找到了节点的大小关系。
      - 如果没有实现，或者两个节点的 key 相等，使用`System.identityHashCode()` 比较。

也就是说，依次使用如下函数比较：

- `HashMap#hash(key.hashCode())` 
- `equals()`
- `Comparable#compareTo()`，如果 key 实现了 `Comparable`。
- `System.identityHashCode()` ,即 `Object#hashCode()` ，在 该类没有覆盖 `hashCode()` 前应该返回的值。

```java
final TreeNode<K,V> putTreeVal(HashMap<K,V> map, Node<K,V>[] tab,
                               int h, K k, V v) {
  Class<?> kc = null;
  boolean searched = false;
  TreeNode<K,V> root = (parent != null) ? root() : this;
  for (TreeNode<K,V> p = root;;) {
    int dir, ph; K pk;
    if ((ph = p.hash) > h)
      dir = -1;
    else if (ph < h)
      dir = 1;
    else if ((pk = p.key) == k || (k != null && k.equals(pk)))
      return p;
    else if ((kc == null &&
              (kc = comparableClassFor(k)) == null) ||
             (dir = compareComparables(kc, k, pk)) == 0) {
      if (!searched) {
        TreeNode<K,V> q, ch;
        searched = true;
        if (((ch = p.left) != null &&
             (q = ch.find(h, k, kc)) != null) ||
            ((ch = p.right) != null &&
             (q = ch.find(h, k, kc)) != null))
          return q;
      }
      dir = tieBreakOrder(k, pk);
    }

    TreeNode<K,V> xp = p;
    if ((p = (dir <= 0) ? p.left : p.right) == null) {
      Node<K,V> xpn = xp.next;
      TreeNode<K,V> x = map.newTreeNode(h, k, v, xpn);
      if (dir <= 0)
        xp.left = x;
      else
        xp.right = x;
      xp.next = x;
      x.parent = x.prev = xp;
      if (xpn != null)
        ((TreeNode<K,V>)xpn).prev = x;
      moveRootToFront(tab, balanceInsertion(root, x));
      return null;
    }
  }
}

static int tieBreakOrder(Object a, Object b) {
  int d;
  if (a == null || b == null ||
      (d = a.getClass().getName().
       compareTo(b.getClass().getName())) == 0)
    d = (System.identityHashCode(a) <= System.identityHashCode(b) ?
         -1 : 1);
  return d;
}
```

# 构造函数

```java
static final float DEFAULT_LOAD_FACTOR = 0.75f;
static final int MAXIMUM_CAPACITY = 1 << 30;

public HashMap(int initialCapacity, float loadFactor) {
  if (initialCapacity < 0)
    throw new IllegalArgumentException("Illegal initial capacity: " + initialCapacity);
  if (initialCapacity > MAXIMUM_CAPACITY)
    initialCapacity = MAXIMUM_CAPACITY;
  if (loadFactor <= 0 || Float.isNaN(loadFactor))
    throw new IllegalArgumentException("Illegal load factor: " + loadFactor);
  this.loadFactor = loadFactor;
  this.threshold = tableSizeFor(initialCapacity);
}

public HashMap(int initialCapacity) {
  this(initialCapacity, DEFAULT_LOAD_FACTOR);
}

public HashMap() {
  this.loadFactor = DEFAULT_LOAD_FACTOR; // all other fields defaulted
}
```

- 构造函数中只指定了 `loadFactor` 和 初始容量。

- `threshold` 有两个作用。
  - 在创建存储 Node 的数组 `table` 没有创建时，存储**初始**容量，即随后 `table` 的长度。
  - 在 `table` 创建后，如果负载超过 `threshold`，进行 `resize` 扩容。阈值的初始值和变化请看 resize() 源码分析处。
- `table` 的长度为 2 的次幂。

## tableSizeFor()

如果构造函数指定了 `initialCapacity`，调用 `tableSizeFor` 转化为大于等于 `initialCapacity` 的最小的 2 的次幂。

`tableSizeFor` 思想是，把 cap 从最右边的 1 开始到最低位，都变为 1。再加 1，即为 2 的 次幂。

```java
static final int tableSizeFor(int cap) {
  int n = cap - 1; // 如果 cap 本来就是 2 的次幂，不减 1，经过下面操作后，会变为 2 * cap。
  
  // n 的最右边至少有一个 1，例如 1XXXX。
  n |= n >>> 1;	// 把 n 的前 2 位都变为 1，例如 11XXX。
  n |= n >>> 2; // 把 n 的前 4 位都变为 1，例如 1111X。
  n |= n >>> 4; // 类似
  n |= n >>> 8;	// 类似
  n |= n >>> 16; // 最高位 32 个 1。
  // 以上操作，把 n 转化为每位都是 1。
  
  // n + 1 后只有最高位为1，为 2 的次幂
  return (n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1;
}
```

## putMapEntries()

```java
public HashMap(Map<? extends K, ? extends V> m) {
  this.loadFactor = DEFAULT_LOAD_FACTOR;
  putMapEntries(m, false);
}

final void putMapEntries(Map<? extends K, ? extends V> m, boolean evict) {
    int s = m.size();
    if (s > 0) { 
        if (table == null) { // pre-size // 构造函数调用 putMapEntries，一定会进入这个分支。
            float ft = ((float)s / loadFactor) + 1.0F;
            int t = ((ft < (float)MAXIMUM_CAPACITY) ?
                     (int)ft : MAXIMUM_CAPACITY);
            if (t > threshold)
                threshold = tableSizeFor(t);
        }
        else if (s > threshold) // putAll 调用时，可能会进入这个分支。
            resize();
        for (Map.Entry<? extends K, ? extends V> e : m.entrySet()) {
            K key = e.getKey();
            V value = e.getValue();
            putVal(hash(key), key, value, false, evict);
        }
    }
}
```

除此之外，还有第 4 个构造函数，把给定 Map 的每个元素使用 `putVal()` 加入本 HashMap 中。

构造函数调用 `putMapEntries`，只会进入第一个分支。把初始容量变为 `要加入元素个数 / loadFactor`。

# resize()

## 触发时机

- 初始化` table` 时。
- 元素个数超过 `threshold` 时进行扩容。

## 源码分析

```java
final Node<K,V>[] resize() {
    Node<K,V>[] oldTab = table;
    int oldCap = (oldTab == null) ? 0 : oldTab.length;
    int oldThr = threshold;
    int newCap, newThr = 0;
  
  	// 已存在 table，是触发扩容进入 resize() 函数的。
    if (oldCap > 0) {
        if (oldCap >= MAXIMUM_CAPACITY) {
            threshold = Integer.MAX_VALUE;
            return oldTab;
        }
      	// table.length 加倍
      	// threshold 加倍
        else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                 oldCap >= DEFAULT_INITIAL_CAPACITY)
            newThr = oldThr << 1; // double threshold
    }
  	// 构造函数指定了 initialCapacity，存储在 threshold中，所以 > 0。
    else if (oldThr > 0) 
        newCap = oldThr;
    // 构造函数没有指定 initialCapacity。
  	else {              
        newCap = DEFAULT_INITIAL_CAPACITY; // 1 << 4
        newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
    }
  
  	// 从此，threshold 存储的是扩容的阈值。
  	// 初始大小为 table.length * loadFactor
    if (newThr == 0) {
        float ft = (float)newCap * loadFactor;
        newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                  (int)ft : Integer.MAX_VALUE);
    }
    threshold = newThr;
  
  
    @SuppressWarnings({"rawtypes","unchecked"})
    Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
    table = newTab;
    if (oldTab != null) {
        for (int j = 0; j < oldCap; ++j) { // 把每个元素插入新的 table 中。
            Node<K,V> e;
            if ((e = oldTab[j]) != null) {
                oldTab[j] = null;
                if (e.next == null)	// 如果存储桶里只有一个元素，直接插入对应位置。
                    newTab[e.hash & (newCap - 1)] = e;
                else if (e instanceof TreeNode) // 如果存储桶里是红黑树，拆分树。TODO
                    ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                else { // 如果存储桶里是链表，拆分链表
                    Node<K,V> loHead = null, loTail = null;
                    Node<K,V> hiHead = null, hiTail = null;
                    Node<K,V> next;
                    do {
                        next = e.next;
                        if ((e.hash & oldCap) == 0) {
                            if (loTail == null)
                                loHead = e;
                            else
                                loTail.next = e;
                            loTail = e;
                        }
                        else {
                            if (hiTail == null)
                                hiHead = e;
                            else
                                hiTail.next = e;
                            hiTail = e;
                        }
                    } while ((e = next) != null);
                    if (loTail != null) {
                        loTail.next = null;
                        newTab[j] = loHead;
                    }
                    if (hiTail != null) {
                        hiTail.next = null;
                        newTab[j + oldCap] = hiHead;
                    }
                }
            }
        }
    }
    return newTab;
}
```

### 第一部分 容量和阈值的变化

容量指 `table` 数组的长度。

阈值指 `threshold`。

两种情况

- 在 `table` 还没有创建时：
  - 容量为构造函数传入的初始容量或默认值（16）。
  - 阈值为容量 * loadFactor。
- 在 `table` 还没有创建时：
  - 容量为原容量的 2 倍。
  - 阈值为原阈值的 2 倍。

### 第二部分 红黑树拆分

拆成两棵红黑树。

并检查每一棵红黑树的节点数，如果节点数少于阈值，转成树。

### 第三部分 链表拆分

- `e.hash & oldCap == 0` 的节点组成 low 链表，存入 `newTable[j]` 位置。
- `e.hash & oldCap != 0` 的节点组成 high 链表，存入 `newTable[j + oldCap]` 位置。

原因如下。

假设 `oldCap = 16 = 0b1_0000`，存储在 `j` 处的元素是 `j = e.hash & 0b0_1111`。

扩容后 `newCap = 16 * 2 = 0b10_0000`，节点的下标是 `e.hash & 0b1_1111`。

- 如果 `e.hash` 的第 5 位是 0，则 `e.hash & 0b1_1111` 与 `e.hash & 0b0_1111` 相等，所以还存储在 `j` 处。
- 如果 `e.hash` 的第 5 位是1，则 `e.hash & 0b1_1111` 与 `(e.hash & 0b0_1111) + 0b1_0000`相等，所以存储在 `j + oldCap` 处。

# put()

```java
public V put(K key, V value) {
    return putVal(hash(key), key, value, false, true);
}

final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
               boolean evict) {
    Node<K,V>[] tab; Node<K,V> p; int n, i;
  	// 构造函数没有初始化 table。
  	// 因此，第一次 put，会在这里初始化 table。
    if ((tab = table) == null || (n = tab.length) == 0)
        n = (tab = resize()).length;
  	// 如果对应的桶里没有元素，直接加进去。
    if ((p = tab[i = (n - 1) & hash]) == null)
        tab[i] = newNode(hash, key, value, null);
    else {
        Node<K,V> e; K k;
      
      	// 找这个桶里，与 key 相同的元素，赋值给 e。
      	// 没有找到，就创建一个节点，插入到桶的链表/红黑树里，e 为 null。
      
        if (p.hash == hash && // 如果桶里只有一个元素，且 key 相同，找到 e 了。
            ((k = p.key) == key || (key != null && key.equals(k))))
            e = p;
        else if (p instanceof TreeNode) // 在红黑树里找。
            e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
        else { // 在链表里找。
            for (int binCount = 0; ; ++binCount) {
                if ((e = p.next) == null) { // 整个链表都没有找到，创建一个节点加入链表。
                    p.next = newNode(hash, key, value, null);
                    if (binCount >= TREEIFY_THRESHOLD - 1) // 节点数目大于等于 8 个，扩容或变为红黑树。
                        treeifyBin(tab, hash);
                    break;
                }
                if (e.hash == hash &&	// 在链表中找到 e 了。
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    break;
                p = e;
            }
        }
      
        if (e != null) { // 如果找到了之前存在的节点，更新值。
            V oldValue = e.value;
            if (!onlyIfAbsent || oldValue == null) // onlyIfAbsent 只有putIfAbsent调用时是true
                e.value = value;
            afterNodeAccess(e); // 只有 LinkedHashMap 时有用，HashMap 中 afterNodeAccess 是空的。
            return oldValue;
        }
    }
  
  	// 只有新建节点会到这里
    ++modCount;
    if (++size > threshold) // 如果元素数目多于阈值，扩容。
        resize();
    afterNodeInsertion(evict); // 只有 LinkedHashMap 时有用
    return null;
}

final void treeifyBin(Node<K,V>[] tab, int hash) {
  int n, index; Node<K,V> e;
  if (tab == null || (n = tab.length) < MIN_TREEIFY_CAPACITY) // 如果 table 长度小于64，则扩容。
    resize();
  else if ((e = tab[index = (n - 1) & hash]) != null) { // 否则桶中链表转化为树。
    TreeNode<K,V> hd = null, tl = null;
    do {
      TreeNode<K,V> p = replacementTreeNode(e, null);
      if (tl == null)
        hd = p;
      else {
        p.prev = tl;
        tl.next = p;
      }
      tl = p;
    } while ((e = e.next) != null);
    if ((tab[index] = hd) != null)
      hd.treeify(tab);
  }
}
static final int MIN_TREEIFY_CAPACITY = 64;
```

- 第一次 put，会创建 table 数组。
- 在桶的链表/红黑树里找节点，没找到，就创建一个，插入链表/红黑树中。
- 如果链表插入节点后，链表长度多于 8 个，进行扩容或转化为红黑树。
- 如果找到了节点，并且不是 `putIfAbsent()` 调用的，更新节点的值。
- 如果是新增了节点，查看节点数目是否多于阈值，进行扩容。

# get()

```java
public V get(Object key) {
    Node<K,V> e;
    return (e = getNode(hash(key), key)) == null ? null : e.value;
}

final Node<K,V> getNode(int hash, Object key) {
    Node<K,V>[] tab; Node<K,V> first, e; int n; K k;
    if ((tab = table) != null && (n = tab.length) > 0 && // table 没有初始化，或对应桶是空的，返回。
        (first = tab[(n - 1) & hash]) != null) {
        if (first.hash == hash && // 桶中第一个节点是否是对应的key。
            ((k = first.key) == key || (key != null && key.equals(k))))
            return first;
        if ((e = first.next) != null) {
            if (first instanceof TreeNode) // 如果是红黑树，在红黑树里找。第一个节点是红黑树的根节点。
                return ((TreeNode<K,V>)first).getTreeNode(hash, key);
            do {	// 在链表中找。
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    return e;
            } while ((e = e.next) != null);
        }
    }
    return null;
}
```

- table 没有初始化、或对应的桶是空的，返回。
- 查看桶中第一个节点的key是否匹配。
- 没有匹配，去桶的红黑树或链表中去找。