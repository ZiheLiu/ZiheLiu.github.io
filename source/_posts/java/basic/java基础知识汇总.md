---
title: Java 基础知识汇总
date: 2020-03-04 14:48:52
tags:
	- java
categories:
	- java	
	- basic
typora-root-url: ../../../
---

# Java 基础知识汇总

Java 区分大小写。

源代码的文件名必须与公共类的名字相同。

main 方法必须声明为 public。

允许在运行时确定数组的长度。

javabeans TODO

## 3. 继承

“is-a” 关系是继承的一个明显特征。

继承使用 `extends` 来表示。

Java 只存在公有继承。

超类不能使用子类定义的方法和域。

子类方法不能访问超类的 `private` 域，只能通过公有的接口访问。

### 3.1.` super` 关键字

#### 作用

- 调用超类的方法。
- 调用超类的构造函数。只能出现在子类函数的第一条语句。

#### 与 `this` 的不同

- 不是对象的引用。
- 用来指示编译器调用超类方法的关键字。

### 3.2. 多态与动态绑定

#### 3.2.1 多态

##### 定义

一个对象变量可以指示多种实际类型的现象（一个 `ClassA` 变量可以引用 `ClassA` 类对象、它的子类对象）。

##### 原理

- 虚拟机知道变量实际引用的对象类型，可以正确地调用相应的方法。
- TODO

##### 子类数组

**子类**数组的引用可以隐式转化成**父类**数组的引用。

但是数组会记住创建它们的真实元素类型，只允许类型兼容的引用存储到数组中。

```java
Child[] children = new Child[10];
Parent[] parents = children; // 合法
parents[0] = new Parent(); // throw ArrayStoreException
```

##### 强制类型转换

超类转化为子类，需要进行强制类型转换。

如果转换失败，throw ClassCastException。

在转换之前，最好用 `item instanceof ClassType` 来检测。



#### 3.2.2 动态绑定

- 在运行时能够自动选择调用哪个方法的现象。
- 动态绑定是默认的处理方式。



#### 3.2.3 方法调用的过程

调用 `item.func(param)`：

1. 编译器获得所有可能的候选方法。根据对象的声明类型（`item` 的实际类型）、方法名，找到类的所有 `func` 方法、超类的所有 `public` 修饰的 `func` 方法。
2. 匹配提供的参数类型（除了一个隐式参数）。TODO
3. 如果是 `private`、`static`、`final` 方法，是**静态绑定**，无法被继承，编译器知道应用调用的确切方法。否则，编译器采用动态绑定的方式生成一条调用 `func` 的指令。
4. 在程序运行时，如果 `item.func` 是动态绑定方式调用的，虚拟机找到与 `item` 所引用对象的实际类型 `D` 最合适的类的 `func`方法。首先看类 `D` 是否有对应的方法，再看超类，以此类推。
   为了节约开销，虚拟机会为每一个类生成方法表，不需要每次都进行动态搜索。

### 3.3 重载 overload 和 覆写 override

函数的签名：方法名  + 参数列表。注意，**返回值**不属于签名！

因此，

- 重载时，不可以声明两个函数名、参数列表都相同，只有返回值不同的函数。
- 覆写时，子类的函数返回值可以与父类不同，但是必须是兼容的。即子类的返回值必须是父类的返回值类型、或其子类型。否则，编译错误。

对于覆写的方法，加上 `@override` 修饰，如果这个方法没有真的覆写父类的方法，编译错误。

覆写方法的访问权限要大于等于原方法的权限。（本来是public，子类的方法也必须是public）。

覆写方法可以指定更加严格的返回类型。

###3.4 final 类和方法，内联

##### final 关键字

`final` 修饰的方法，子类不可以覆盖它。

`final` 修饰的类，不能有子类继承它。

- 其中方法自动被 `final` 修饰，但是域不会。

##### 内联

- 如果一个方法没有被覆盖并且很短，编译器会优化它，进行内联。

- 虚拟机的即时编译器，检测方法没有真正的覆盖、并且被频繁调用，会对它进行内联处理。
- TODO 即时编译器？

优点：

- 使用分支转移（函数调用），会扰乱 CPU 预取指令的策略。

### 3.5 abstract 关键字

`abstract` 修饰的方法，不需要有函数体。

有抽象方法的类，必须用 `abstract` 修饰。但是 `abstract` 修饰的类，不一样要有抽象方法。

子类如果没有实现全部的抽象方法，必须也用 `abstract` 修饰类。

抽象类可以包含具有函数体的方法。

### 3.6 Object 类

- 是所有类的超类。

- ```java
  Object obj = new Employee("name", 100);
  ((Employee) obj).func();
  ```

- 数组也扩展了 Object 类。

#### equals 函数

Java语言规范要求 `equals` 方法具有 5 个特性：

1. 自反性
2. 对称性
3. 传递性
4. 一致性
5. 对于非空引用 `item`，`item.equals(null)` 应该返回false。

编写 `equals` 的步骤：

1. 参数命名为 `otherOject`，之后转化后的命名为 `other`。
2. 检测 `this` 与 `otherObject` 是否引用了相同的对象。
3. 检测 `otherObject` 是否为 null。
4. 比较 `this` 与 `otherObject` 是否属于同一个类，用 `instanceof` 或 `getClass()`。
5. 将 `otherObject` 转化为具体的类型对象 `other`。
6. 进行域的比较。

```java
  @Override
  public boolean equals(Object otherObject) {
    if (this == otherObject) {
      return true;
    }
    if (otherObject == null) {
      return false;
    }
    // 或者 不同子类可以进行比较
    // if (!(otherOBject instance of ParentClass)) {
    // 	return false;
    // }
    if (this.getClass() != otherObject.getClass()) {
      return false;
    }
    
    Item other = (Item) otherObject;
    return num == other.num;
  }
```

#### hashCode 函数

`equals` 函数 与 `hashCode` 函数的定义必须一致。

- 如果 `x.equals(y)== true` ，那么 `x.hashCode()== y.hashCode()`。

静态方法 `Object.hashCode(item)`，如果 item 是 null，会返回0，否则调用 item 的 hashCode 方法。

静态方法 `Object.hash(item1, item2, ...)` 可以提供多个参数。

类的 `hashCode()` 方法，默认返回引用的地址。

#### toString 函数

对象与一个字符串通过 `+` 连接起来，会自动调用对象的 `toString()` 函数。

`"" + x` 与 `x.toString()` 等同，但是 `x` 是基本类型时，不能用 `x.toString()`。

默认方法，输出 类型 + 散列码。

### 4. 包装类、自动装箱和拆箱

#### 包装类

8 个类：Long、Integer、Short、Byte，Double、Float，Boolean，Void。

这些类的值是 `final` 的，不可变。

类也是 `final` 的，不可被继承。

因为是类，所以进行算数运算时，可能 throw NullPointerException。

#### 自动拆箱和装箱

发生在编译期间，编译器补全拆箱和装箱的代码。

#### 性能差

- `ArrayList<Integer>` 的每个值包在一个对象里，性能远低于 `int[]`。

- 算数运算可能进行频繁的拆箱、装箱。

  ```java
  Integer n = 1;
  Double x = 2.0;
  System.out.println(ok ? n : x); // n 先拆箱，再转为 double，再装箱为 Double
  ```

#### 相等判断

包装类在比较时，最好使用 `equals`。

因为 `==` 可能无法得到预期的结果。

boolean/byte/char <= 127/-128-127的short和int，会被包装到固定的对象中。此时，值相等，用 `==` 判断也会相等。

### 5. 参数数量可变方法

函数最后一个参数可以是 `ClassName... args`，编译器会替换为 `ClassName[] args`。`func(item1, item2)` 会替换为 `func(ClassName[] {item1, item2})`。

### 6. 枚举类

```java
public enum Size {
  SMALL,
  MEDIUM,
  LARGE;
}
```

实际上，`Size` 是定义了一个类。`SMALL`、`MEDIUM`、`LARGE` 是类 `Size` 的三个实例。比较枚举的值，使用 `==` 即可。

因为它是类，所以可以添加 `private` 构造函数。
```java
public enum Size {
  SMALL(0),
  MEDIUM(1),
  LARGE(2);
  
  private int value;
  private Size(int value) {
    this.value = value;
  }
  
  public int getValue() {
    return value;
  }
}
```

## 4. 接口

### 4.0 相关函数

#### Arrays.sort

调用 `Arrays.sort(items)`，数组中的元素必须实现了 `Comparable<T>`。

调用 `Arrays.sort(items, comparator)`，提供一个实现了 `Comparator<T>` 接口的对象即可。

#### instanceof

`instanceof` 可以检查一个对象是否实现了一个接口。

#### Cloneable

`Object.clone` 方法

- 是浅拷贝。
- 如果要深拷贝，需要再克隆每个非基本变量域。
- 是 `protected` 的，所以如果要克隆对象，需要 `override` 这个方法，调用 `super.clone()`。
  - 并且需要实现 `Cloneable` 接口。否则会在运行时抛出 `CloneNotSupportedException`。

`Cloneable` 是一个空的接口，**标记接口**，作用是允许在类型查询中使用 `instanceof`。



### 4.1 接口的特性

#### 接口的方法和域

接口的所有**方法**自动属于 `public`。

接口的所有**域**都会自动属于 `public static final`。所以接口只能有常量，不能有实例域。

Java SE 8 中，接口可以有**静态方法**。避免使用伴随类（例如，Collection 接口 + Collections 抽象类）。

使用 `defualt` 修饰方法，可以为接口方法提供默认实现。实现类可以不实现这些方法。



#### 默认方法冲突规则

1. 类实现的**两个接口**有相同签名的方法
   - 只要其中一个方法有默认实现，就会编译错误。
2. 类实现的**接口**和继承的**超类**有相同签名的方法
   - 超类的方法优先，接口方法的默认实现被忽略。



#### 继承和实现相关

接口可以被其他接口继承 `extends`。

##### 为什么有了抽象类，还需要接口？

一个类可以实现多个接口，但是只能继承一个类。



## 5. lambda 表达式

### 5.1 语法

代码块 + 必须传入代码块的变量的规范。

```java
(String first, String second) -> {
  return first.length() - second.length();
}
```

如果可以推导出参数的类型，可以省略类型。

如果参数只有一个，并且可以推导类型，可以省略 `()`。

不用指定返回类型，从上下文推导出的。

如果代码块只有一个语句，可以省略`{}`、`return` 关键字。

```java
(first, second) -> first.length() - second.length();
```

### 5.2 函数式接口

#### 定义

对于只有一个**抽象方法**的接口，需要接口的对象时，可以提供一个 lambda 表达式。

在这个对象上调用这个方法时，会执行 lambda 表达式的体。



可以在这个接口定义时，增加 `@FunctionalInterface` 注解。好处：

- 如果有多于一个的抽象方法，会编译错误。
- javadoc 会指出这个借口是函数式接口。

#### Object 相关

注意，lambda 表达式只能赋值给一个接口变量，不能复制给 `Object`（因为 `Object` 不是一个接口）。

#### BiFunction

`java.util.function` 提供了很多通用函数接口，例如 `BiFunction<T, U, R>` 描述参数类型为 `T `和 `U` 而且返回类型为 `R` 的函数。

```java
BiFunction<T, U, R> func = (first, second) -> first.length() - second.length();
```

但是使用接口和函数式接口的地方不能传递一个 `BiFunction`。

### 5.3 方法引用

形如 `System.out::println` 是一个方法引用，等价于 lambda 表达式 `x -> System.out.println(x)`。

共 3 类：

- `object::instanceFunc`：`x -> object.instanceFunc(x)`。
- `ClassName::staticFunc`：`x -> ClassName.staticFunc(x)`。
- `ClassName::instanceFunc`：`(x, y) -> x.instanceFunc(y)`。

关于 `this` 和 `super`：

- `this::instanceFunc`：`x -> this.instanceFunc(x)`。
- `super::instanceFunc`：`x -> super.instanceFunc(x)`。

多个同名的重载函数时，根据上下文找到最合适的函数。

### 5.4 构造器引用

与方法引用类似，方法名为 `new`。例如 `Person::new`。

可以有数组的构造器引用，`int[]::new`。

Java 无法构造泛型 `T` 的数组，即不可以使用 `new T[10]`。`stream.toArray(Person[]::new)` 可以用类似泛型的方式构造数组。

### 5.5 变量作用域

引用的外部变量，必须是不可变的，相当于必须是 `final` 的，但是不用使用 `final` 修饰符。

lambda 表达式的体与嵌套块有相同的作用域。

- 因此不能声明和嵌套块中的同名变量。

- 体中的 `this` 指的是嵌套块所在对象，不是函数式接口这个对象。

  ```java
  public class Application {
    public void init() {
      ActionListener linstener = event -> {
        // this 是 Application，而不是 ActionListener
        Systme.out.println(this.toString());
      }
    }
  }
  ```

### 5.6 Comparator

`Comparator.comparing(Person::getName)` 提供“键提取函数”。

`Comparator.comparing(Person::getFirstName).thenComparing(Person::getSecondName)` 可以和thenComparing 串起来。

comparing 和 thenComparing 可以提供第二个参数，是一个 Comparator 对象。`Comparator.comparing(Person::getName, (x, y) -> x > y)`。



`Comparator` 的 `nullFirst` 和 `nullSecond` 方式使装饰器，接收一个 Comparator 对象，遇到 null 时，标记为小于正常值 或 大于正常值。 



## 6. 内部类

内部类是定义在另一个类中的类。

3 个优点：

- 内部类可以访问该类定义所在作用域的数据，包括 `private` 的数据。
- 内部类可以对同一个包中的其他类隐藏起来。
- 匿名内部类实现回调函数方便。

### 6.1 内部类访问外部对象

#### 编译器转化形式

内部类之所以可以访问创建它的外部类对象的数据域，是因为内部类的对象有一个隐式的外部类对象的引用。是编译器加上去的。

```java
public class TalkingClock {
  private boolean keep;
  
  public class TimePrinter implements ActionListener {
    @override
    public void actionPerformed(ActionEvent event) {
      if (keep) {
        // ...
      }
    }
  }
  
  public void start() {
    TimePrinter t = new TimePrinter();
    new Timer(1000, t).start();
  }
}
```

会被编译器添加内部类的关于 `outer` 的构造函数、生成内部类对象时添加 `this` 参数：

```java
public class TalkingClock {
  private boolean keep;
  
  class TimePrinter implements ActionListener {
    TalkingClock outer;
    TimePrinter(TalkingClock outer) {
      this.outer = outer;
    }
    
    @override
    public void actionPerformed(ActionEvent event) {
      if (keep) {
        // ...
      }
    }
  }
  
  public void start() {
    TimePrinter t = new TimePrinter(this);
    new Timer(1000, t).start();
  }
}
```

实际上，内部类会被翻译成 `TalkingClock$TimePrinter` 的类形式。

- 自己无法编写类似内部类的形式，因为内部类可以访问外部类的 `private` 域，但是自己定义的不可以。
- 如何实现的访问私有域的：`TalkingClock.access$0(outer)`。

#### 内外访问方式

在内部类中使用外部类的 `this` 对象，可以用 `OuterClass.this` 的形式。

在外部类作用域之外，使用 `OuterClass.InnerClass` 来访问这个内部类，如果它是 `public` 的。

#### 规则限制

内部类的静态域必须是 `final` 的。因为每一个外部类的对象，都有一个单独的内部类实例，为了保证“静态域只有一个实例”。

内部类不能有 `static` 方法。

### 6.2 局部内部类

#### 规则限制

局部内部类不能用 `public` 或 `private` 修饰，因为作用域被限定在声明这个局部内部类的块中。

可以访问外部函数中的变量，但是必须是类 `final` 的（Java SE 8 之前必须显式声明为 `final` 的）。

#### 原理

- 会把访问的外部函数的变量，进行备份。其实是，添加到了构造函数中。

#### 绕过外部变量的 final 限制

要改变外部变量 `int num = 1`，定义为 `int[] num = {1}` 长度为 1 的数组即可。

### 6.3 匿名内部类

构造函数调用的右小括号后面跟一个开大括号，正在定义的就是匿名内部类。

#### 双括号初始化

```java
ArrayList<String> friends = new ArrayList<>();
friends.add("a");
friends.add("b");
invite(friends);

// 可以简写为
invite(new ArrayList<>() {
  { add("a"), add("b")} // 对象构造块
});
```

#### 在静态方法中获取 class

```java
new Object(){}.getClass().getEnclosingClass();
```

### 6.4 静态内部类

#### 定义

用 `static ` 修饰内部类，则内部类内部没有外部类的隐式引用，不能使用外部类的域。

只是为了把类隐藏在另一个类内部。

注意，只有内部类可以使用 `static` 修饰。

#### 规则限制

静态内部类可以有静态域和静态方法。

#### 接口中的内部类

声明在接口中的内部类自动为 `public` 和 `static`。



## 7. 异常

### 7.1 异常层次结构

- Throwable 类
  - Error 类：Java 运行时系统的内部错误和资源好紧错误。
  - Exception 类
    - RuntimeException 类：程序错误导致的异常。
    - 其他类：程序本身没有问题。

`Error` 和 `RuntimeException`是 uncheckd 异常，其他异常是 checked 异常。



### 7.2 声明 unchecked 异常

在方法左花括号前面声明可能抛出的异常

```java
public FileInputStream(String name) throws FileNotFoundException;
```

不需要声明 `Error` 和 `RuntimeException`。

函数中调用方法声明的异常，必须被 catch 或声明。

#### override 中的异常声明

- 子类 `override` 父类的方法，声明的异常不能比父类方法的声明更通用，只能更具体。

- 如果父类方法没有声明异常，子类方法也不可以。



### 7.3 抛出异常

```java
throw new MyException(); // 必须是 Throwable 的子类
```



### 7.4 创建异常类

定义派生于 `Exception` 类或其子类的类。

```java
public class MyException extends Exception {
  public MyException() {}
  public MyException(String msg) {
    // 存储在 throwable.detailMessage 域中
    // throwable.getMessage() 方法会返回它
    super(msg); 
  }
}
```



### 7.5 捕获异常

#### 执行流程

- 执行 `try` 中代码，`try` 代码块是否抛出了某个 `catch` 中的异常。
  - 抛出了，则不再执行try 中代码，执行 `catch` 中代码。
  - 没有抛出，跳过 `catch` 代码。
- 执行 `finally` 中代码。

#### 捕获语法

```java
try {
  
} catch (FileNotFoundException e) {
  // ...
} catch (MyException e) {
  // ...
}

// Java Se 7 中，可以一个 catch 语句捕获多个异常类型
try {
  
} catch (FileNotFoundException | MyException e) {
  // ...
}
```

捕获的异常变量隐含为 `final`。

可以在 `catch` 中再次抛出捕获的异常或者新的异常。

#### finally 中的 return

`finally` 中的 return 会覆盖掉 `try` 和 `catch` 中的 return。

#### 带资源的 try 语句

Java SE 7 中新增。

```java
try (Resource1 res1 = ...; Resource2 res2 = ...; Resource3 res3 = ...;) {
	// ...
}
```

`Resource` 必须是实现了 `AutoCloseable` 接口的类。



编译器会自动补全相关的调用 `res.close()` 的方法。

并且，如果 `try` 块抛出了异常 `e1`，`close` 的时候也抛出了异常 `e2`，会 `e1.addSuppressed(e1)`，抛出 `e1`。



## 8. 泛型

### 8.1 定义

#### 泛型类

诸如 `T` 的类型变量，用尖括号 `<>` 括起来，放在类名的后面。

```java
public class Pair<T, U> {
  private T first;
  private U second;
  
  public Pair(T first, U second) {
    this.first = first;
    this.second = second;
  }
  
  public T getFirst() {
    return first;
  }
  
  public void setFirst(T first) {
    this.first = first;
  }
}
```

#### 泛型方法

泛型方法可以定在普通类中。

定义时，类型变量放在修饰符后面，返回类型的前面。

```java
class ArrayUtils {
	public static <T> T getMiddle(T... items) {
    return items[item.length / 2];
  }
}
```

使用时，在方法名前面加上 `<具体类型>`。

```java
String middle = ArrayUtils.<String>getMiddle("A", "B", "C");
```

大多数情况，可以省略 `<具体类型>`，由编译器推断出。

```java
String middle = ArrayUtils.getMiddle("A", "B", "C");
```

#### 类型变量的限定

使用 `extends` 关键字，`<T extends BoundingType>` 来限定类型变量 `T` 是 `BoundingType` 的子类型。

##### 多个类型限定

一个类型变量可以有多个限定，用 `&` 分隔开，如下：

```java
<T extends Comparable & Serializable>
```

限定中，最多只能有一个类，必须是限定列表的第一个。（原因见泛型的类型擦除）。



### 8.2 类型擦除和虚拟机

Java 要求旧版本的代码可以运行在新版本的 JVM 上，所以选择了在编译时擦除泛型的类型。

#### 擦除规则

定义时

- 编译后的字节码，只保留**原始类型**，即删除类型参数后的类名和方法名。
- **擦除**类型变量，替换为限定类型。
  - 没有限定，用 `Object` 替换。
  - 有多个限定，用第一个替换类型变量。

调用时

- 编译器在必要的位置插入强制类型转换。

#### 桥方法

方法的类型参数在擦除时，关于 `overload` 和 `override` 会出现冲突。

```java

class Food {}

class Fruit extends Food {}

class Pair<T> {
  T first;
  T second;

  public T getFirst() {
    return first;
  }

  public void setFirst(T first) {
    System.out.println(first.getClass());
    this.first = first;
  }

  public T getSecond() {
    return second;
  }

  public void setSecond(T second) {
    this.second = second;
  }
}

class FruitPair extends Pair<Fruit> {
  @Override
  public void setSecond(Fruit second) {
    super.setSecond(second);
  }

  @Override
  public Fruit getSecond() {
    return super.getSecond();
  }
}

// example
FruitPair fruitPair = new FruitPair();
Pair<Fruit> pair = fruitPair;
pair.setSecond(new Fruit());
```

##### 写方法

对于写方法 `setSecond`，存在自己定义的 `setSecond(Fruit second)`，父类 `Pair` 的方法是 `setSecond(Object second)`。这两个方法的签名是不同的，无法进行 `override` 以及多态。

例如，在上述 example 中，调用 `pair.setSecond` 时。因为 `pair` 的声明是 `Pair`，不是 `FruitPair`，所以无法看到 `setSecond(Fruit second)` 方法，只能看到 `setSecond(Object second)`。这样会调用父类的 setSecond 方法，违反了程序的本意。

因此，编译器会在类 `FruitPair` 中生成一个 `setSecond(Object second)` **桥方法**来完成覆写和多态：

```java
public void setSecond(Object second) {
  setSecond((Fruit) second);
}
```

##### 读方法

对于读方法 `getSecond`，存在自己定义的 `Fruit getSecond()` 和 父类的方法 `Object getSecond()`。可以看到这两个方法的签名是相同的。

虚拟机中用参数类型和返回类型确定函数。虽然编写代码时，签名不包括返回值，但是编译器可能产生两个仅返回值不同的函数，虚拟机可以正确处理。

为了完成覆盖和多态，子类 `FruitPair` 会生成一个 `Object getSecond()` 方法：

```java
public Object getSecond() {
  return getSecond();
}
```

##### 泛型场景之外的桥方法

子类的方法可以返回更严格的返回类型，也是用桥方法实现的。



### 8.3 约束与局限

#### 1). 泛型的类型变量不能是基本类型 `List<int>`

因为擦除类型后，会用 `Object` 替换， `Object` 不能存储 `int`。

#### 2). 运行时类型查询只能找到原始类型

```java
Pair<Fruit> fruitPair;
Pair<String> strPair;
assert(fruitPair.getClass() == strPair.getClass()); // 因为都是 Pair 类的对象

fruitPair instanceof Pair<Fruit>; // 编译错误
(Pair<Fruit>) fruitPair; // 可以，编译警告
```

#### 3). 不能 new 参数化类型的数组 `new Pair<Fruit>[10]`

```java
Pair<Fruit>[] pairs = new Pair<Fruit>[10]; // 编译错误

Pair<Fruit>[] pairs = (Pair<Fruit>[]) new Pair<?>[10]; // 可以，编译警告
```

#### 4). 在函数参数中 new 参数化类型的数组

```java
public static <T> void addAll(T... items) {}
addAll(new Pair<Fruit>(), new Pair<Fruit>());
```

相比规则 3)，放松了规则，只会得到警告。

#### 5). 不能实例化类型变量 `new T()`

```java
new T(); // 编译错误
new T[10]; // 编译错误
T.class; // 编译错误
```

因为如果运行，类型擦除后，都会变为 `new Object()`，和程序本意不符。

`Class` 类是泛型类，`String.class` 返回了 `Class<String>` 的唯一实例。

#### 6). 不能实例化类型变量的数组 `new T[10]`

因为如果允许，这样 new 出来的是 `Object[]`，无法转化成其他类的对象。

如果只是内部使用数组，可以如下，强制类型转化 `E[]` 是假象。

```java
public class ArrayList<E> {
  private E[] elements;
  public ArrayList() {
    elements = (E[]) new Object[10];
  }
}
```

但是如果用这种方法，返回了` E[]`，是不可以的，因为它是用 `new Object[]`构建的。所以编译不会报错，在执行到 AAA 处时，报出 ClassCastException。

```java
public class ArrayList<E> {
  private E[] elements;
  public ArrayList() {
    elements = (E[]) new Object[10];
  }
  E[] getList() {
    return elements;
  }
} 

ArrayList<Integer> nums;
Integer[] nums2 = nums.getList(); // AAA
```

#### 7). 泛型类不能有静态的类型变量

如果允许，`Pair<String>` 和 `Pair<Integer>` 会生成两个静态域，不符合语义。

```java
class Pair<T> {
  static T item;
}
```

#### 8). 擦除后的冲突

擦除类型后，如果有冲突是不可以的。

如下，这样会生成 `equals(Object)` 与超类 `Object` 的方法冲突。

```java
class Pair<T> {
  public boolean equals(T value) {}
}
```

一个类不能同时成为两个接口类型的子类，这两个接口类型是同一个接口的不同参数化。

如下，`Fruit` 实现了 `Comparable<Food>` 和 `Comparable<Fruit>`。

```java
class Food implements Comparable<Food> {}
class Fruit extends Food implements Comparable<Fruit> {}
```



### 8.4 通配符

[知乎-通配符](https://www.zhihu.com/question/20400700/answer/117464182)

#### 8.4.1 继承关系

`Pair<Fruit>` 与 `Pair<Food>` 没有什么关系，他们都是 `Pair` 的子类。

```java
Pair<Food> pair = new Pair<Fruit>(); // 编译错误
```



#### 8.4.2 上界通配符 extends

`Pair<? exnteds Food>` 表示这里最通用可以是 `Food`，即 `Food` 和 `Food` 的子类。

```java
Pair<? extends Food> pair = new Pair<Fruit>();
```



不能存，只能取。

因为编译器只知道这里是 Food 和 Food 的子类，不知道具体是什么类的对象。编译器实际上会变为 `Pair<#CAP1>`，用 #CAP1 来占位。



#### 8.4.3 下界通配符

`Pair<? super Fruit>` 表示这里最具体可以使 `Fruit`，即 `Fruit` 和 `Fruit` 的超类。



可以存，取的时候只能赋值给 `Object`。

因为这里规定了下界，既然元素是 Fruit 的基类，那么存入 Fruit 的子类都可以。

但是因为模糊了上界，所以只能赋给 Object。



