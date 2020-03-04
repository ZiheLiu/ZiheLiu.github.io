---
title: Enum
date: 2020-03-04 14:48:52
tags:
	- java
categories:
	- java
	- basic
typora-root-url: ../../../
---

# Enum

## 1. 定义

### 基础

test

使用 `enum`  关键字，类似于定义类的方式。

```java
enum Fruit {
    ORANGE,
    BANANA;
}
```

### 自定义构造函数

可以自定义构造函数，传递额外的信息。注意 `Fruit` 类的第一个语句必须是枚举变量的定义。

```java
enum Fruit {
    ORANGE(10),
    BANANA(3);
    
    private int price;
    
    enum Fruit(int price) {
        this.price = price;
    }
}
```

## 2. 原理

编译器会把 `enum` 类转化为 `Enum<E extends Enum<E>>` 的子类。

例如，上述的 `Fruit` 枚举类，反编译后如下：

```java
final class Fruit extends Enum
{

    public static Fruit[] values()
    {
        return (Fruit[])$VALUES.clone();
    }

    public static Fruit valueOf(String name)
    {
        return (Fruit)Enum.valueOf(Fruit, name);
    }

    private Fruit(String name, int ordinal, int price)
    {
        super(name, ordinal);
        this.price = price;
    }

    public int getPrice()
    {
        return price;
    }

    public static final Fruit ORANGE;
    public static final Fruit BANANA;
    
    private int price;
    private static final Fruit $VALUES[];

    static 
    {
        ORANGE = new Fruit("ORANGE", 0, 10);
        BANANA = new Fruit("BANANA", 1, 3);
        $VALUES = (new Fruit[] {
            ORANGE, BANANA
        });
    }
}
```

主要进行的工作：

1. 在构造函数中添加 `name` 、`ordinal` 。
2. 每个枚举量是静态常量，会调用 1. 中的构造函数。其中 `name` 是枚举变量名、 `ordinal` 从0开始编号。
3. 枚举量列表存储在静态常量 `$VALUES` 中。
4. 为 `Fruit` 类添加 `final` 修饰符。

## 3. 常用函数

根据反编译后的类、超类 `Enum` ，可以发现如下的常用函数：

1. `static Fruit valueOf(String name)`  返回和传入 `name` 匹配的枚举量。如果不能匹配，抛出 `IllegalArgumentException` 。
2. `String name()` 、`String toString()`  返回 `name` 。
3. `int ordinal()`  返回 `ordinal` 。
4. `static Fruit[] values()`  返回 枚举量数组。
5. `int compareTo(Fruit other)`  返回 `ordinal - other.ordinal` 。

比较两个枚举量是否相等，用 `==` 即可，即比较两个枚举常量的地址是否相同。  
当然，用 `equals` 也可以，因为 `Enum` 的 `equals` 函数就是直接 `return this == other` 。

## 4. switch 中的枚举量

### 定义

可以在 `switch` 中使用枚举量。注意 `case` 中不需要写 `Fruit.XXXX` ，直接 `XXXX` 即可。

```java
Fruit fruit = Fruit.BANANA;
switch (fruit) {
  case BANANA:
    System.out.println("banana");
    break;
  case ORANGE:
    System.out.println("orange");
    break;
}
```

### 原理

编译器会把代码进行转换，创建一个 SwitchMap 类 `Main$1` ，使用 `ordinal` 进行比较。

反编译后如下：

```java
// Main$1.class
// $FF: synthetic class
class Main$1 {

   // $FF: synthetic field
   static final int[] $SwitchMap$Fruit = new int[Fruit.values().length];

   static {
      try {
         $SwitchMap$Fruit[Fruit.BANANA.ordinal()] = 1;
      } catch (NoSuchFieldError var2) {
         ;
      }

      try {
         $SwitchMap$Fruit[Fruit.ORANGE.ordinal()] = 2;
      } catch (NoSuchFieldError var1) {
         ;
      }

   }
}

// Main.class
import Main.1;

class Main {
   public static void main(String[] args) {
      Fruit fruit = Fruit.BANANA;
      switch(1.$SwitchMap$Fruit[fruit.ordinal()]) {
      case 1:
         System.out.println("banana");
         break;
      case 2:
         System.out.println("orange");
      }
   }
}
```

## 5. 枚举类的序列化

### 引用

- [枚举的序列化如何实现](https://github.com/hollischuang/toBeTopJavaer/blob/master/basics/java-basic/enum-serializable.md)

### 类似枚举类的序列化问题

在 Java 没有枚举类的时候，一般用如下的方法实现枚举类。

```java
public class Fruit {
  public static final ORANGE = new Fruit(0);
  public static final BANANA = new Fruit(1);
  
  private int value;
  
  private Fruit(int value) {
    this.value = value;
  }
}
```

例子如下。在反序列化的时候，虽然构造函数是私有的，但是反序列化仍然会创造一个新的对象。
这个对象的 `value` 也是 0，但是没有办法使用 `==` 与 `Fruit.ORANGE` 进行比较了。

```java
objectOutputStream.write(Fruit.ORANGE);
objectOutputStream.close();
Fruit orange = (Fruit) objectInputStream.read();
aseertt(orange == Fruit.ORANGE); // Error
```

### 枚举类的序列化

枚举类在序列化的时候，对于 `Enum` 子类的对象，只会把 `name` 属性输出。

在反序列化的时候，会调用 `Enum valueof(String name)` 方法对读取的 `name` 进行查找，而不会创建新的 枚举对象。

并且类 `Enum` 把 方法 `readObject` 和 `readObjectNoData` 都设为了私有，这样 `enum` 类无法自定义序列化过程。

