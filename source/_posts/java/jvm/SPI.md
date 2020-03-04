---
title: SPI 机制
date: 2020-03-04 14:48:52
tags:
	- java
	- jvm
categories:
	- java
	- jvm
typora-root-url: ../../../
---

# SPI 机制

## 0. 引用

- [Java SPI机制：ServiceLoader实现原理及应用剖析](https://juejin.im/post/5d2db85d6fb9a07ea7134408)
- [Java SPI机制详解](https://juejin.im/post/5af952fdf265da0b9e652de3)
- [从Dubbo内核-SPI聊聊双亲委派机制](https://juejin.im/post/5c959901e51d456d8054c635#heading-2)

## 1. 介绍

Service Provider Interfaces (SPI)，即服务提供接口。是 JDK 内置的**服务发现和管理**的机制。

### 作用

- 它可以**动态**的为某个接口**寻找服务实现**，有点类似 IOC(Inversion of Control) 的思想，将装配的控制权移到**程序之外**，在模块化设计中这个机制尤其重要。

- 解耦 服务具体实现 和服务使用，使程序的**可扩展性**大大增强，甚至可插拔。

### 服务、服务提供方、服务使用方

- **服务** 一般是系统定义好的接口，
- **服务提供方**提供具体的服务实现、并向系统注册服务，
- **服务使用方**通过查找发现服务、而不是直接引用导入具体的服务，达到与服务提供方的分离。
  - 这样可以把想使用的服务提供方写在配置文件中，不需要修改代码就可以完成服务提供方的更改。

## 2. 实现难点

由于服务**提供者**没有被服务**使用者**引用，因此对服务**提供者**进行类加载的任务需要服务**定义者**来完成。但是服务定义者的类加载器一般是服务提供者类加载器的祖先，默认的**双亲委派**类加载机制使得服务定义者无法完成对服务提供者的类加载。

例如 JDK 中定义了 `JDBC` 服务，而 MySQL、PostgreSQL 提供了具体的实现。`JDBC` 的服务接口定义在 `java.sql` 中，由引导类加载器进行的加载。而具体的服务实现类在 Classpath 下，应该由系统类加载器进行加载。显然，使用 JDBC 的引导类加载器是无法对这些具体服务进行加载的。

因此，在 JDBC 4.0 以前，在连接数据库的时候，需要首先使用 `Class.forName("com.mysql.jdbc.Driver")` 来在服务使用者这里显式的加载服务实现者。

## 3. 解决方法

为了解决这个问题，需要使**父类**加载器去**请求子类**加载器去执行类加载任务，这样就要破坏双亲委派类加载机制。

因此，引入了**线程上下文类加载器**，线程的上下文类加载器默认都是系统类加载器。这样，服务定义者就可以使用这个线程上下文类加载器完成服务提供者的类加载。

## 4. 使用方法

`java.util.ServiceLoader` 实现了上述的方案。

在 `Classpath/resources/META-INF/services/MyService的接口全名` 文件中写入多个服务提供者的类全名（以行为单位），使用 `ServiceLoader` 就可以去加载这些服务提供者。

```java
ServiceLoader<MyService> loader = ServiceLoader.load(MyService.class);
for (MyService serviceImpl : loader) {
  serviceImpl.func();
}
```

### JDBC 例子

#### DriverManager

在 JDBC 中就使用了 `ServiceLoader` 去加载具体的驱动。

`DriverManager` 在类初始化时，使用 `ServiceLoader` 去加载了 `Driver.class` 的实现类。

```java
public class DriverManager {
    static {
        loadInitialDrivers();
        println("JDBC DriverManager initialized");
    }
    private static void loadInitialDrivers() {
        String drivers;
        try {
            drivers = AccessController.doPrivileged(new PrivilegedAction<String>() {
                public String run() {
                    return System.getProperty("jdbc.drivers");
                }
            });
        } catch (Exception ex) {
            drivers = null;
        }
        // If the driver is packaged as a Service Provider, load it.
        // Get all the drivers through the classloader
        // exposed as a java.sql.Driver.class service.
        // ServiceLoader.load() replaces the sun.misc.Providers()

        AccessController.doPrivileged(new PrivilegedAction<Void>() {
            public Void run() {

                ServiceLoader<Driver> loadedDrivers = ServiceLoader.load(Driver.class);
                Iterator<Driver> driversIterator = loadedDrivers.iterator();

                try{
                    while(driversIterator.hasNext()) {
                        driversIterator.next();
                    }
                } catch(Throwable t) {
                // Do nothing
                }
                return null;
            }
        });

        // ...
    }
}
```

####  服务提供者

在 `mysql-connector-java-5.1.45.jar` 中，META-INF/services目录下会有一个名字为java.sql.Driver的文件：

```
com.mysql.jdbc.Driver
com.mysql.fabric.jdbc.FabricMySQLDriver
```

在 `postgresql-42.2.2.jar` 中，META-INF/services目录下会有一个名字为java.sql.Driver的文件：

```
org.postgresql.Driver
```

这样在使用 JDBC 的时候，直接获取连接就可以了，不需要额外加载对应的驱动类。

```java
String url = "jdbc:mysql://localhost:3306/test";
Connection conn = DriverManager.getConnection(url,username,password);
```

## 5. 具体实现

- 使用 `ServiceLoader.load(MyService.class)` 方法，会把当前线程的上下文类加载器存储在 `loader` 中。

- `ServiceLoader#iterator()` 返回一个内置的迭代器。
- `iterator#hasNext()` 会去读取所有 resources 文件夹中 META-INF/services/serviceName 文件。
- `iterator#next()` 用 `loader` 去加载每一个 `hadNext()` 读取出来的类。

```java
// 默认使用上下文类加载器。
public static <S> ServiceLoader<S> load(Class<S> service) {
    ClassLoader cl = Thread.currentThread().getContextClassLoader();
    return ServiceLoader.load(service, cl);
}
public static <S> ServiceLoader<S> load(Class<S> service, ClassLoader loader){
    return new ServiceLoader<>(service, loader);
}

private ServiceLoader(Class<S> svc, ClassLoader cl) {
    service = Objects.requireNonNull(svc, "Service interface cannot be null");
    loader = (cl == null) ? ClassLoader.getSystemClassLoader() : cl;
    acc = (System.getSecurityManager() != null) ? AccessController.getContext() : null;
    reload();
}

public Iterator<S> iterator();

private static final String PREFIX = "META-INF/services/";
// hasNext() 最终调用 hasNextService()
private boolean hasNextService() {
    if (nextName != null) {
        return true;
    }
    if (configs == null) {
        try {
            String fullName = PREFIX + service.getName();
            if (loader == null)
              	// 找到所有 resources 文件夹中 META-INF/services/serviceName 文件。
                configs = ClassLoader.getSystemResources(fullName);
            else
                configs = loader.getResources(fullName);
        } catch (IOException x) {
            fail(service, "Error locating configuration files", x);
        }
    }
  	// 遍历每一个 config 文件。
    while ((pending == null) || !pending.hasNext()) {
        if (!configs.hasMoreElements()) {
            return false;
        }
        pending = parse(service, configs.nextElement());
    }
  	// 存储下一个要加载的类名。
    nextName = pending.next();
    return true;
}
// next() 最终调用 nextService()
private S nextService() {
    if (!hasNextService())
        throw new NoSuchElementException();
    String cn = nextName;
    nextName = null;
    Class<?> c = null;
    try {
      	// 使用构造函数传入的 loader，加载 nextName。
        c = Class.forName(cn, false, loader);
    } catch (ClassNotFoundException x) {
        fail(service,
             "Provider " + cn + " not found");
    }
    if (!service.isAssignableFrom(c)) {
        fail(service,
             "Provider " + cn  + " not a subtype");
    }
    try {
        S p = service.cast(c.newInstance());
        providers.put(cn, p);
        return p;
    } catch (Throwable x) {
        fail(service,
             "Provider " + cn + " could not be instantiated",
             x);
    }
    throw new Error();          // This cannot happen
}
```