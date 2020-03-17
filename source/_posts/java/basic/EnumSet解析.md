---
title: EnumSet 解析
date: 2020-03-17 20:51:00
tags:
	- java
categories:
	- java
	- basic
typora-root-url: ../../../
---

# 简介

`EnumSet` 存储某个枚举类的集合。由于枚举类的对象是固定的，因此内部使用位图来记录每个具体对象是否存在。

# EnumSet.noneOf()

1. 获取枚举类的对象数组，即 `elementType#values`。

2. 把位图都清零。

   - 如果对象个数不多于 64。使用 `RegularEnumSet` 子类，用一个 long 存下位图。

   - 否则，使用 `JumboEnumSet` 子类，用一个 long 数组存储位图。

```java
// EnumSet
public static <E extends Enum<E>> EnumSet<E> noneOf(Class<E> elementType) {
    // 获取枚举类的对象数组，即 elementType#values。
    Enum<?>[] universe = getUniverse(elementType);
    if (universe == null)
        throw new ClassCastException(elementType + " not an enum");

    if (universe.length <= 64)
        return new RegularEnumSet<>(elementType, universe);
    else
        return new JumboEnumSet<>(elementType, universe);
}

// RegularEnumSet
private long elements[];
RegularEnumSet(Class<E>elementType, Enum<?>[] universe) {
    super(elementType, universe);
}

// JumboEnumSet
private long elements = 0L;
JumboEnumSet(Class<E>elementType, Enum<?>[] universe) {
    super(elementType, universe);
    elements = new long[(universe.length + 63) >>> 6];
}
```

# EnumSet#add()

使用 `e.ordinal()` 作为位图的 ID。

- `RegularEnumSet`
  - 直接把 ID 对应位置设为 1 即可。
- `JumboEnumSet`
  1. 找到要修改的 long 数组元素的下标。
  2. 把对应元素对应位置设为 1 即可。

```java
// EnumSet
final void typeCheck(E e) {
    Class<?> eClass = e.getClass();
    if (eClass != elementType && eClass.getSuperclass() != elementType)
        throw new ClassCastException(eClass + " != " + elementType);
}

// RegularEnumSet
public boolean add(E e) {
    typeCheck(e);

    long oldElements = elements;
    elements |= (1L << ((Enum<?>)e).ordinal());
    return elements != oldElements;
}

// JumboEnumSet
public boolean add(E e) {
    typeCheck(e);

    int eOrdinal = e.ordinal();
    // eOrdinal / 64
    int eWordNum = eOrdinal >>> 6;

    long oldElements = elements[eWordNum];
    // 相当于 1L << (eOrdinal % 64)
    elements[eWordNum] |= (1L << eOrdinal);
    boolean result = (elements[eWordNum] != oldElements);
    if (result)
        size++;
    return result;
}
```

#EnumSet.allOf()

1. 调用 `EnumSet.noneOf()`。
2. 把位图所有位置设为 1。

```java
// EnumSet
public static <E extends Enum<E>> EnumSet<E> allOf(Class<E> elementType) {
    EnumSet<E> result = noneOf(elementType);
    result.addAll();
    return result;
}

// RegularEnumSet
void addAll() {
    if (universe.length != 0)
        elements = -1L >>> -universe.length;
}

// JumboEnumSet
void addAll() {
    for (int i = 0; i < elements.length; i++)
        elements[i] = -1;
    elements[elements.length - 1] >>>= -universe.length;
    size = universe.length;
}
```

# 位运算

`EnumSet` 中涉及了两个巧妙的位运算用法。

1. `1L << 68` 相当于 `1L << (68 % 64)`，右移、int 都类似。
2. 想要得到一个二进制后 `num` 位都是 1 的数字。用 `-1 >>> -num` 即可。原理如下：
   - -1 的二进制是 `0xffffffff`，即所有位都为 1。
   - 右移 `-num` 相当于右移 `-num % 32`。
   - 所以，最后相当于 `0xffffffff >>> (32 - num)`。