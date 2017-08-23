---
title: Scala中'_'的用途
date: 2017-08-23 15:08:59
categories: 技术
tags:
    - Scala
---
# 用于替换Java的等价语法

由于大部分的Java关键字在Scala中拥有了新的含义，所以一些基本的语法在Scala中稍有变化。
<!-- more -->

## 导入通配符

'*' 在Scala中是合法的方法名，所以导入包时要使用 '_' 代替。

```scala
//Java
import java.util.*;

//Scala
import java.util._
```

## 类成员默认值

Java中类成员可以不赋初始值，编译器会自动帮你设置一个合适的初始值：

```java
class Foo{
     //String类型的默认值为null
     String s;
}
```

而在Scala中必须要显式指定，如果你比较懒，可以用_让编译器自动帮你设置初始值：

```scala
class Foo{
    //String类型的默认值为null
    var s: String = _
}
```

该语法只适用于类成员，而不适用于局部变量。

## 可变参数

Java声明可变参数如下：

```java
public static void printArgs(String ... args){
    for(Object elem: args){
        System.out.println(elem + " ");
    }
}
```

调用方法如下：

```java
 //传入两个参数
printArgs("a", "b");
//也可以传入一个数组
printArgs(new String[]{"a", "b"});
```

在Java中可以直接将数组传给printArgs方法，但是在Scala中，你必须要明确的告诉编译器，你是想将集合作为一个独立的参数传进去，还是想将集合的元素传进去。如果是后者则要借助下划线：

```scala
printArgs(List("a", "b"): _*)
```

## 类型通配符

Java的泛型系统有一个通配符类型，例如List<?>，任意的List<T>类型都是List<?>的子类型，如果我们想编写一个可以打印所有List类型元素的方法，可以如下声明：

```java
public static void printList(List<?> list){
    for(Object elem: list){
        System.out.println(elem + " ");
    }
}
```

对应的Scala版本为：

```scala
def printList(list: List[_]): Unit ={
   list.foreach(elem => println(elem + " "))
}
```

# 模式匹配

## 默认匹配

```scala
str match{
    case "1" => println("match 1")
    case _   => println("match default")
}
```

## 匹配集合元素

```scala
//匹配以0开头，长度为三的列表
expr match {
  case List(0, _, _) => println("found it")
  case _ =>
}

//匹配以0开头，长度任意的列表
expr match {
  case List(0, _*) => println("found it")
  case _ =>
}

//匹配元组元素
expr match {
  case (0, _) => println("found it")
  case _ =>
}

//将首元素赋值给head变量
val List(head, _*) = List("a")
```

# Scala特有语法

## 访问Tuple元素

```scala
val t = (1, 2, 3)
println(t._1, t._2, t._3)
```

## 简写函数字面量（function literal）

如果函数的参数在函数体内只出现一次，则可以使用下划线代替：

```scala
val f1 = (_: Int) + (_: Int)
//等价于
val f2 = (x: Int, y: Int) => x + y

list.foreach(println(_))
//等价于
list.foreach(e => println(e))

list.filter(_ > 0)
//等价于
list.filter(x => x > 0)
```

## 定义一元操作符

在Scala中，操作符其实就是方法，例如1 + 1等价于1.+(1)，利用下划线我们可以定义自己的左置操作符，例如Scala中的负数就是用左置操作符实现的：

```scala
-2
//等价于
2.unary_-
```

## 定义赋值操作符

我们通过下划线实现赋值操作符，从而可以精确地控制赋值过程：

```scala
class foo {
    def name = { "foo" }
    def name_=(str: String) {
        println("set name " + str)
    }
    val m = new Foo()
    m.name = "Foo" //等价于: m.name_=("Foo")
}

```

## 定义部分应用函数（partially applied function）

我们可以为某个函数只提供部分参数进行调用，返回的结果是一个新的函数，即部分应用函数。因为只提供了部分参数，所以部分应用函数也因此而得名。

```scala

def sum(a: Int, b: Int, c: Int) = a + b + c
val b = sum(1, _: Int, 3)
b: Int => Int = [<]function1[>]
b(2) //6 
```

## 将方法转换成函数

Scala中方法和函数是两个不同的概念，方法无法作为参数进行传递，也无法赋值给变量，但是函数是可以的。在Scala中，利用下划线可以将方法转换成函数：

```scala
//将println方法转换成函数，并赋值给p
val p = println _  
//p: (Any) => Unit
```

# 小结

下划线在大部分的应用场景中是以语法糖的形式出现的，可以减少击键次数，并且代码显得更加简洁。




# 下划线的用途
```scala
1、存在性类型：Existential types
def foo(l: List[Option[_]]) = ...

2、高阶类型参数：Higher kinded type parameters
case class A[K[_],T](a: K[T])

3、临时变量：Ignored variables
val _ = 5

4、临时参数：Ignored parameters
List(1, 2, 3) foreach { _ => println("Hi") }

5、通配模式：Wildcard patterns
Some(5) match { case Some(_) => println("Yes") }
match {
     case List(1,_,_) => " a list with three element and the first element is 1"
     case List(_*)  => " a list with zero or more elements "
     case Map[_,_] => " matches a map with any key type and any value type "
     case _ =>
 }
val (a, _) = (1, 2)
for (_ <- 1 to 10)

6、通配导入：Wildcard imports
import java.util._

7、隐藏导入：Hiding imports
// Imports all the members of the object Fun but renames Foo to Bar
import com.test.Fun.{ Foo => Bar , _ }

// Imports all the members except Foo. To exclude a member rename it to _
import com.test.Fun.{ Foo => _ , _ }

8、连接字母和标点符号：Joining letters to punctuation
def bang_!(x: Int) = 5

9、占位符语法：Placeholder syntax
List(1, 2, 3) map (_ + 2)
_ + _   
( (_: Int) + (_: Int) )(2,3)

val nums = List(1,2,3,4,5,6,7,8,9,10)

nums map (_ + 2)
nums sortWith(_>_)
nums filter (_ % 2 == 0)
nums reduceLeft(_+_)
nums reduce (_ + _)
nums reduceLeft(_ max _)
nums.exists(_ > 5)
nums.takeWhile(_ < 8)

10、偏应用函数：Partially applied functions
def fun = {
    // Some code
}
val funLike = fun _

List(1, 2, 3) foreach println _

1 to 5 map (10 * _)

//List("foo", "bar", "baz").map(_.toUpperCase())
List("foo", "bar", "baz").map(n => n.toUpperCase())

11、初始化默认值：default value
var i: Int = _

12、作为参数名：
//访问map
var m3 = Map((1,100), (2,200))
for(e<-m3) println(e._1 + ": " + e._2)
m3 filter (e=>e._1>1)
m3 filterKeys (_>1)
m3.map(e=>(e._1*10, e._2))
m3 map (e=>e._2)

//访问元组：tuple getters
(1,2)._2

13、参数序列：parameters Sequence 
_*作为一个整体，告诉编译器你希望将某个参数当作参数序列处理。例如val s = sum(1 to 5:_*)就是将1 to 5当作参数序列处理。
//Range转换为List
List(1 to 5:_*)

//Range转换为Vector
Vector(1 to 5: _*)

//可变参数中
def capitalizeAll(args: String*) = {
  args.map { arg =>
    arg.capitalize
  }
}

val arr = Array("what's", "up", "doc?")
capitalizeAll(arr: _*)
```

这里需要注意的是，以下两种写法实现的是_完全不一样_的功能：

```scala
foo _               // Eta expansion of method into method value

foo(_)              // Partial function application
```

Example showing why foo(_) and foo _ are different:

```scala
trait PlaceholderExample {
  def process[A](f: A => Unit)

  val set: Set[_ => Unit]

  set.foreach(process _) // Error 
  set.foreach(process(_)) // No Error
}
```

In the first case, process _ represents a method; Scala takes the polymorphic method and attempts to make it monomorphic by filling in the type parameter, but realizes that there is no type that can be filled in for A that will give the type (_ => Unit) => ? (Existential _ is not a type).

In the second case, process(_) is a lambda; when writing a lambda with no explicit argument type, Scala infers the type from the argument that foreach expects, and _ => Unit is a type (whereas just plain _ isn't), so it can be substituted and inferred.

This may well be the trickiest gotcha in Scala I have ever encountered.