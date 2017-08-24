---
title: Java8 Lambda 表达式学习
date: 2017-08-24 09:39:01
categories: 技术
tags:
    - Java
---

# 背景

匿名内部类并不是一个好的选择，因为：

1. 语法过于冗余
2. 匿名类中的this和变量名容易使人产生误解
3. 类型载入和实例创建语义不够灵活
4. 无法捕获非final的局部变量
5. 无法对控制流进行抽象

<!-- more -->

**java8解决：**

- 通过提供更简洁的语法和局部作用域规则，Java SE 8彻底解决了问题1和问题2
- 通过提供更加灵活而且便于优化的表达式语义，Java SE 8绕开了问题3
- 通过允许编译器推断变量的“常量性”(finality)，Java SE 8减轻了问题4带来的困扰

不过，Java SE 8的目标并非解决所有上述问题。因此捕获可变变量(问题4)和非局部控制流(问题5)并不在Java SE 8的范畴之内。(尽管我们可能会在未来提供对这些特性的支持)

# 函数式接口(Functional interfaces)

尽管匿名内部类有着种种限制和问题，但是它有一个良好的特性，它和Java类型系统结合的十分紧密:
**每一个函数对象都对应一个接口类型**。之所以说这个特性是良好的，是因为:

- 接口是Java类型系统的一部分
- 接口天然就拥有其运行时表示(Runtime representation)
- 接口可以通过Javadoc注释来表达一些非正式的协定(contract)，例如，通过注释说明该操作应可交换(commutative)

# lambda 表达式

lambda表达式的语法由参数列表、箭头符号->和函数体组成。
函数体既可以是一个表达式，也可以是一个语句块：

- 表达式：表达式会被执行然后返回执行结果。
- 语句块：语句块中的语句会被依次执行，就像方法中的语句一样——
    - return 语句会把控制权交给匿名方法的调用者
    - break和continue只能在循环中使用
- 如果函数体有返回值，那么函数体内部的每一条路径都必须返回值
表达式函数体适合小型lambda表达式，它消除了return关键字，使得语法更加简洁。

# 目标类型(Target typing)

需要注意的是，函数式接口的名称并不是lambda表达式的一部分。那么问题来了，对于给定的lambda表达式，它的类型是什么？答案是：它的类型是由其上下文推导而来。

lambda表达式对目标类型也是有要求的。编译器会检查lambda表达式的类型和目标类型的方法签名(method signature)是否一致。当且仅当下面所有条件均满足时，lambda表达式才可以被赋给目标类型T:

- T是一个函数式接口
- lambda表达式的参数和T的方法参数在数量和类型上一一对应
- lambda表达式的返回值和T的方法返回值相兼容(Compatible)
- lambda表达式内所抛出的异常和T的方法throws类型相兼容

当lambda的参数只有一个而且它的类型可以被推导得知时，该参数列表外面的括号可以被省略：

```java
FileFilter java = f -> f.getName().endsWith(".java");
button.addActionListener(e -> ui.dazzle(e.getModifiers()));
```

# 目标类型的上下文(Contexts for target typing)

之前我们提到lambda表达式智能出现在拥有目标类型的上下文中。下面给出了这些带有目标类型的上下文：

- 变量声明
- 赋值
- 返回语句
- 数组初始化器
- 方法和构造方法的参数
- lambda表达式函数体
- 条件表达式(? :)
- 转型(Cast)表达式

# 词法作用域(Lexical scoping)

这里的作用域 同 方法作用域

```java
public class Hello {

    'Runnable r1 = () -> { System.out.println(this);}'
    'Runnable r2 = () -> { System.out.println(toString());}'

    public String toString() {  return "Hello, world"; }

    public static void main(String... args) {
        new Hello().r1.run();
        new Hello().r2.run();
    }
}
```


# 变量捕获(Variable capture)
- 在Java SE 7中，编译器对内部类中引用的外部变量(即捕获的变量)要求非常严格：如果捕获的变量没有被声明为final就会产生一个编译错误。
- lambda表达式对值封闭，对变量开放。
      + 原因：
        1. 因为这样的lambda表达式很容易引起race condition。除非我们能够强制(最好是在编译时)这样的函数不能离开其当前线程，但如果这么做了可能会导致更多的问题
        2. 可以使用更好的方式来实现同样的效果：使用规约(reduction)。java.util.stream包提供了各种通用的和专用的规约操作(例如sum、min和max)
- 如果操作符具有可结合性(associative)，那么规约操作就可以容易的被并行化

```java
int sum = 0;
list.forEach(e -> { sum += e.size(); }); // Illegal, close over values

List<Integer> aList = new List<>();
list.forEach(e -> { aList.add(e); }); // Legal, open over variables

int sum = list.stream().mapToInt(e -> e.size()).sum();
```

等价于

```java
int sum = list.stream().mapToInt(e -> e.size()).reduce(0 , (x, y) -> x + y);
```

# 方法引用(Method references)

lambda表达式允许我们定义一个匿名方法，并允许我们以函数式接口的方式使用它

```java
class Person {
    private final String name;
    private final int age;

    public int getAge() { 
        return age; 
    }
    public String getName() {
        return name; 
    }
    ...
}
Person[] people = ...
Comparator<Person> byName = Comparator.comparing(p -> p.getName());
Arrays.sort(people, byName);
```

在这里我们可以用方法引用代替lambda表达式：

```java
Comparator<Person> byName = Comparator.comparing(Person::getName);
```

# 方法引用的种类(Kinds of method references)

方法引用有很多种，它们的语法如下：

  - 静态方法引用：ClassName::methodName
  - 实例上的实例方法引用：instanceReference::methodName
  - 超类上的实例方法引用：super::methodName
  - 类型上的实例方法引用：ClassName::methodName
  - 构造方法引用：Class::new
  - 数组构造方法引用：TypeName[]::new

# 默认方法和静态接口方法(Default and static interface methods)
Java SE 7时代为一个已有的类库增加功能是非常困难的.具体的说，接口在发布之后就已经被定型，除非我们能够一次性更新所有该接口的实现，否则向接口添加方法就会破坏现有的接口实现。默认方法(之前被称为虚拟扩展方法或守护方法)的目标即是解决这个问题，使得接口在发布之后仍能被逐步演化。
默认方法利用面向对象的方式向接口增加新的行为。它是一种新的方法：接口方法可以是抽象的或是默认的。默认方法拥有其默认实现，实现接口的类型通过继承得到该默认实现(如果类型没有覆盖该默认实现)。此外，默认方法不是抽象方法，所以我们可以放心的向函数式接口里增加默认方法，而不用担心函数式接口的单抽象方法限制。

```java
interface Iterator<E> {
  boolean hasNext();
  E next();
  void remove();

  default void skip(int i) {
    for ( ; i > 0 && hasNext(); i -= 1) next();
  }
}
```

除了默认方法，Java SE 8还在允许在接口中定义静态方法.

# 继承默认方法(Inheritance of default methods)

# 融会贯通(Putting it together)

演示代码：
```java
List<Person> people = new ArrayList<>();
Collections.sort(people, new Comparator<Person>() {
  public int compare(Person x, Person y) {
    return x.getLastName().compareTo(y.getLastName());
  }
})
```

冗余代码实在太多了！
有了lambda表达式，我们可以去掉冗余的匿名类：

```java
Collections.sort(people,(Person x, Person y) -> x.getLastName().compareTo(y.getLastName()));

```

借助Comparator里的comparing方法实现比较操作：

```java
Collections.sort(people, Comparator.comparing((Person p) -> p.getLastName()));
```

在类型推导和静态导入的帮助下，我们可以进一步简化上面的代码：

```java
Collections.sort(people, comparing(p -> p.getLastName()));
```

用方法引用代替它：

```java
Collections.sort(people, comparing(Person::getLastName));
```

List增加默认方法sort()，然后就可以这样调用：

```java
people.sort(comparing(Person::getLastName));;
```

此外，如果我们为Comparator接口增加一个默认方法reversed()(产生一个逆序比较器)，我们就可以非常容易的在前面代码的基础上实现降序排序。

```java
people.sort(comparing(Person::getLastName).reversed());
```

## 名称解释
- Race Condition中文翻译是竞争条件，是指多个进程或者线程并发访问和操作同一数据且执行结果与访问发生的特定顺序有关的现象。换句话说，就是线程或进程之间访问数据的先后顺序决定了数据修改的结果，这种现象在多线程编程中是经常见到的)



## 相关链接

[深入理解Java 8 Lambda(语言篇——lambda，方法引用，目标类型和默认方法)](http://lucida.me/blog/java-8-lambdas-insideout-language-features)

[深入理解Java 8 Lambda(类库篇——Streams API，Collectors和并行)](http://zh.lucida.me/blog/java-8-lambdas-inside-out-library-features)

[深入探索Java 8 Lambda表达式](http://www.infoq.com/cn/articles/Java-8-Lambdas-A-Peek-Under-the-Hood)

[深入浅出 Java 8 Lambda 表达式](http://blog.oneapm.com/apm-tech/226.html)

[Lambda Expressions](https://docs.oracle.com/javase/tutorial/java/javaOO/lambdaexpressions.html)

[Lambda表达式教程](http://blog.csdn.net/ioriogami/article/details/12782141/)

[Java 8 Language Features](http://developer.android.com/preview/j8-jack.html)

[Race Condition现象及产生的原因](http://blog.csdn.net/xingjiarong/article/details/47603813)

