---
title: Java 中用到的设计模式
date: 2020-03-13 15:33:00
tags:
	- java
categories:
	- java
	- basic
typora-root-url: ../../../
---

# 引用

- [图说设计模式](https://design-patterns.readthedocs.io/zh_CN/latest/index.html)
- [在Java的JDK中，我们能学到哪些设计模式？](https://zhuanlan.zhihu.com/p/64062500)
- [设计模式 | 中介者模式及典型应用](https://juejin.im/post/5bd275dc51882529290fe2c5)
- [细看JDBC——分工和桥接模式](https://www.unclewang.info/learn/java/771/)
- [一起学设计模式 - 桥接模式](https://segmentfault.com/a/1190000011931181)
- [设计模式之外观模式（八）](https://www.jianshu.com/p/6f227102858c)
- [String：字符串常量池](https://segmentfault.com/a/1190000009888357)
- [轻松学，Java 中的代理模式及动态代理](https://blog.csdn.net/briblue/article/details/73928350)
- [Java设计模式（六） 代理模式 vs. 装饰模式](http://www.jasongj.com/design_pattern/proxy_decorator/)

# 结构性模式

## 适配器模式

将一个接口适配到另外一个接口上。

```java
java.io.InputStreamReader(InputStream);
java.io.OutputStreamReader(OutputStream);
```

## 桥接模式

将抽象部分与它的实现部分分离，使它们都可以独立地变化。

某个类存在两个独立变化的维度，通过该模式可以将这两个维度分离出来，使**两者可以独立扩展**，让系统更加符合“单一职责原则”。

```
JDBC 提供两套接口，

- 面向数据库厂商的 Driver 接口。
- 面向 JDBC 使用者的 Connection 接口。
```

## 装饰模式

为给一个类增加行为和功能，子类化的一种替代方法。

```java
BufferedInputStream(InputStream);
DataInputStream(InputStream);
netty WrappedByteBuf/CompositeByteBuf
```

## 门面模式

也叫外观模式。为一组组件，接口，抽象或子系统提供简化的接口。

对客户屏蔽子系统组件，降低客户端与子系统的耦合度，提供一个统一的入口。

```
SLF4J 使用外观模式，提供了统一的日志接口。可以使用不同的日志实现类。
java.lang.Class 提供的接口
```

![slf4j](/images/slf4j.png)

## 享元模式

通过共享技术实现相同或相似对象的重用。

```
String 常量池
对象池
```

### 字面量与常量池

下面的例子，分配对应如下四种情况：

1. 分配一个 11 长度的 `char` 数组，并在常量池分配一个由这个 `char` 数组组成的` String`，然后由 `str`` 去引用这个字符串。
2. 用 `str2` 去引用常量池里边的` String`，所以和 `str` 引用的是同一个对象。
3. 生成一个新的` String`，但内部的字符数组引用着 `str1` 内部的字符数组。
4. 生成一个新的` String`，但内部的字符数组引用常量池中的` String`内部的字符数组。

```java
String str1 = "hello,world";
String str2 = "hello,world";
String str3 = new String(str1);
String str4 = new String("hello,world");

str1 == str2; // true
str1 == str3; // false
str3 == str4; // false
```

```java
private final byte[] value;
// 调用这个构造函数。只是字符数组相同，但是是不同的兑现。
public String(String original) {
    this.value = original.value;
    this.coder = original.coder;
    this.hash = original.hash;
}
```

## 代理模式

为其它对象提供一种代理以控制对这个对象的访问。

```java
java.lang.reflect.Proxy;
RPC（例如 Dubbo）框架中，客户端调用的服务接口的对象。代理对象通过远程访问实现调用。
```

### 与装饰模式的区别

![Proxy Pattern class diagram](/images/ProxyPattern.png)

对于接口 `ISubject`、实现类 `ConcreteSubject`、`SubjectProxy`。`SubjectProxy` 指定了含有 `ConcreteSubject` 域 `item`，它的方法实现，都是通过使用 `item` 的方法加上自己的修改实现的。

而装饰模式，装饰类虽然也是继承自 `ISubject`，但是它是对所有 `ISubject` 接口的功能进行拓展，所以它含有 `ISubject` 的域，而不是具体实现类的域。

动态代理见 {% post_link java/basic/Proxy与CGLib  %}

# 行为模式

## 命令模式

将命令包装在对象中，以便可以将其存储、传递到方法中。

使得请求的接受者与发送者之间解耦。

```java
java.lang.Runnable
```

## 中介模式

多个接口的实现类对象之间复杂的交互提取出来，在中介类中完成交互。

```java
// 使得 Runnable 之间如何协调的交互，放在 Executor 和 ExecutorService 中进行。
Executor.execute(Runnable);
ExecutorService.submit(Callable);
```

## 迭代器模式

提供一个统一的方式来访问集合中的对象。

```java
java.util.Iterator;
java.util.Enumeration;
```

## 观察者模式

Netty 的 `DefaultPromise` 实际上实现了观察者模式。

在调用设置结果的函数时，会调用 `notifyListeners()`，来调用 `DefaultPromise#addListerner()` 添加的每一个 Listener。

## 状态模式

具有状态的对象，在外界与对象互动后，内部的状态发生改变。

![/images/State.jpg](/images/State.jpg)

## 策略模式

将一组算法封装成一系列对象。通过调用这些对象可以灵活的改变程序的功能。

```java
java.util.Comparator#compare();
java.lang.Comparable#compareTo();
netty EventExecutorChooserFactory
  - 线程池大小是 2 的倍数，用位运算
  - 否则，用区域运算
```

# 创建模式

## 抽象工厂模式

## 工厂方法模式

## 建造者模式

用于通过定义一个类来简化复杂对象的创建，该类的目的是构建另一个类的实例。

```java
StringBuilder;
```

## 单例模式

```java
java.lang.Runtime.getRuntime();
```

