---
title: Proxy 与 CGLib
date: 2020-03-13 20:10:00
tags:
	- java
categories:
	- java
	- basic
typora-root-url: ../../../
---

# 引用

- [深入理解 RPC 之动态代理篇](https://www.cnkirito.moe/rpc-dynamic-proxy/)
- [Java Proxy和CGLIB动态代理原理](https://www.cnblogs.com/CarpenterLee/p/8241042.html)

# 原理

`java.lang.reflect.Proxy` 与 `CGLib` 都是

1. 通过运行时动态生成字节码（`bytes` 数组形式）。
2. 用 `ClassLoader` 加载生成的字节码进而得到代理类。
3. 通过反射创建代理对象。

# 用法

为了方便，这里写下需要代理的接口和实现类。

```java
interface HelloService {
  void sayHello(String str1, String str2);
}

class HelloServiceImpl implements HelloService {
  @Override
  public void sayHello(String str1, String str2) {
    System.out.println("HelloImpl: " + str1 + str2);
  }
}
```

## JDK Proxy

```java
class JdkProxy implements InvocationHandler {
  final HelloService delegate;

  public JdkProxy(HelloService delegate) {
    this.delegate = delegate;
  }

  @Override
  public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
    if("sayHello".equals(method.getName())) {
      System.out.println("JdkProxy");
    }
    return method.invoke(delegate, args);
  }
  
  public static HelloService createProxy() {
    return  (HelloService) Proxy.newProxyInstance(
      JdkProxy.class.getClassLoader(), // 1. 类加载器
      new Class<?>[] {HelloService.class}, // 2. 代理需要实现的**接口**，可以有多个
      new JdkProxy(new HelloServiceImpl()));// 3. 方法调用的实际处理者
  }
}


HelloService cgLibService = CgLibProxy.createProxy();
cgLibService.sayHello("a", "b");
```

JDK `Proxy` 可以代理**接口**，`invoke()` 方法为方法调用的实际入口。

## CGLib

### 代理接口

```java
class CgLibProxy implements MethodInterceptor {
  final HelloService delegate;

  public CgLibProxy(HelloService delegate) {
    this.delegate = delegate;
  }

  @Override
  public Object intercept(Object obj, Method method, Object[] args, MethodProxy methodProxy) throws Throwable {
    if("sayHello".equals(method.getName())) {
      System.out.println("CgLibProxy");
    }
    return methodProxy.invoke(delegate, args);
  }
  
  public static HelloService createProxy() {
    Enhancer enhancer = new Enhancer();
    enhancer.setCallback(new CgLibProxy(new HelloServiceImpl())); // 实际处理者
    enhancer.setInterfaces(new Class[]{HelloService.class}); // 代理需要实现的接口
    return (HelloService) enhancer.create();
  }
}

HelloService cgLibService = CgLibProxy.createProxy();
cgLibService.sayHello("a", "b");
```

CGLib 可以代理接口，`intercept()` 方法为方法调用的实际入口。

### 代理类

CGLib 除了代理接口，还可以代理一个类。

生成的代理类是被代理类的子类，所以无法代理 `final`、`private` 修饰的方法，并且无法代理含有抽象函数的类。

```java
class CgLibProxyClass implements MethodInterceptor {
  @Override
  public Object intercept(Object delegate, Method method, Object[] args, MethodProxy methodProxy) throws Throwable {
    if("sayHello".equals(method.getName())) {
      System.out.println("CgLibProxy");
    }
    return methodProxy.invokeSuper(delegate, args);
  }

  public static HelloServiceImpl createProxy() {
    Enhancer enhancer = new Enhancer();
    enhancer.setCallback(new CgLibProxyClass()); // 实际处理者
    enhancer.setSuperclass(HelloServiceImpl.class); // 代理需要实现的接口
    return (HelloServiceImpl) enhancer.create();
  }
}
```

# 性能对比

对于代理接口的功能，比较 `JdkProxy` 和 `CgLibProxy`。

- 比较创建代理类的性能。
- 比较执行 10_000_000 次方法的性能。

```java
long time = System.currentTimeMillis();
HelloService jdkService = JdkProxy.createProxy();
time = System.currentTimeMillis() - time;
System.out.println("Create JDK Proxy:" + time + "ms");

time = System.currentTimeMillis();
HelloService cgLibService = CgLibProxy.createProxy();
time = System.currentTimeMillis() - time;
System.out.println("Create CgLib Proxy:" + time + "ms");

for (int i = 0; i < 100; i++) {
  jdkService.sayHello("a", "b");  // warm up
}
time = System.currentTimeMillis();
for (int i = 0; i < 10_000_000; i++) {
  jdkService.sayHello("a", "b");
}
time = System.currentTimeMillis() - time;
System.out.println("Call method JDK Proxy:" + time + "ms");

for (int i = 0; i < 100; i++) {
  cgLibService.sayHello("a", "b");  // warm up
}
time = System.currentTimeMillis();
for (int i = 0; i < 10_000_000; i++) {
  cgLibService.sayHello("a", "b");
}
time = System.currentTimeMillis() - time;
System.out.println("Call method CgLib Proxy:" + time + "ms");
```

结果如下：

```
Create JDK Proxy: 40ms
Create CgLib Proxy: 100ms
Call method JDK Proxy: 57ms
Call method CgLib Proxy: 45ms
```

结果并没有网上说的 CGLib 比 JDK Proxy 快十倍。

- 创建代理类的时间，JDK Proxy 更快一些；

- 执行方法的时间，二者相近。