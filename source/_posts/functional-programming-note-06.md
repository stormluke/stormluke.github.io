---
title: 函数式编程笔记 06
date: 2013-07-01
url: functional-programming-note-06
---

### 函数即对象

Scala中函数值*确实*被当做对象对待。

函数类型`A => B`是类`scala.Function1[A, B]`的缩写，像下面这样定义：

``` scala
package scala
trait Function1[A, B] {
    def apply(x: A): B
}
```

所以说函数是有`apply`方法的对象。

还有`Function2`、`Function3`等等这些traits，可以有更多的参数（目前到22）。

<!-- more -->

### 函数值的展开

类似于下面的匿名函数

``` scala
(x: Int) => x * x
```

会被展开为：

``` scala
{ class AnonFun extends Function[Int, Int] {
    def apply(x: Int) = x * x
  }
  new AnonFun
}
```

或者用*匿名类语法*变得更短：

``` scala
new Function1[Int, Int] {
    def appy(x: Int) = x * x
}
```

### 函数调用的展开

一个函数调用，比如`f(a, b)`，其中`f`是某个类类型值，会被展开成

``` scala
f.apply(a, b)
```

所以

``` scala
val f = (x: Int) => x * x
f(7)
```

的面向对象版翻译就是：

``` scala
val f = new Function1[Int, Int] {
    def apply(x: Int) = x * x
}
f.apply(7)
```

### 函数和方法

注意一个方法，比如：

``` scala
def f(x: Int): Boolean = …
```

自己并不是一个函数值。

不过如果`f`被用在一个期望函数类型的地方，它就会自动被转换为函数值：

``` scala
(x: Int) => f(x)
```

或者是展开的形式：

``` scala
new Function1[Int, Boolean] {
    def apply(x: Int) = f(x)
}
```

这种转换为匿名函数的方法在λ演算中叫做[η变换(eta-expansion)](https://zh.wikipedia.org/wiki/%CE%9B%E6%BC%94%E7%AE%97#.CE.B7-.E5.8F.98.E6.8D.A2)。

### 对象无处不在

纯面向对象语言是一种在其中每个值都是对象的语言。

### 纯Boolean

`Boolean`类型被映射到JVM基本的`boolean`值。

但是也可以把它定义为一个类：

``` scala
package idealized.scala
abstract class Boolean {
    def ifThenElse[T](t: => T, e: => T): T
    def && (x: => Boolean): Boolean = ifThenElse(x, False)
    def || (x: => Boolean): Boolean = ifThenElse(True, x)
    def unary_! : Boolean = ifThenElse(False, True)
    def == (x: Boolean): Boolean = ifThenElse(x, x.unary_!)
    def != (x: Boolean): Boolean = ifThenElse(x.unary_!, x)
    …
}
```

`True`和`False`这两个常量呢？这样：

``` scala
package idealized.scala
object True extends Boolean {
    def ifThenElse[T](t: => T, e: => T) = t
}
object False extends Boolean {
    def ifThenElse[T](t: => T, e: => T) = e
}
```

### 练习：定义一个类来表示非负整数

``` scala
abstract class Nat {
  def isZero: Boolean
  def predecessor: Nat
  def successor = new Succ(this)
  def + (that: Nat): Nat
  def - (that: Nat): Nat
}
object Zero extends Nat {
  def isZero = true
  def predecessor = throw new Error("0.predecessor")
  def + (that: Nat) = that
  def - (that: Nat) = if (that.isZero) this else throw new Error("negative number")
}
class Succ(n: Nat) extends Nat {
  def isZero = false
  def predecessor = n
  def + (that: Nat) = new Succ(n + that)
  def - (that: Nat) = if (that.isZero) this else n - that.predecessor
}
```

简单解释一下：

假设现在有6个数（[0], [1], [2], [3], [4], [5]），按上面的就表示为

``` scala
Zero
Succ(Zero)
Succ(Succ(Zero))
Succ(Succ(Succ(Zero)))
Succ(Succ(Succ(Succ(Zeror))))
Succ(Succ(Succ(Succ(Succ(Zero)))))
```

则计算 `[2] + [3]` 的过程为

``` scala
[2] + [3]
Succ([1] + [3])
Succ(Succ([0] + [3]))
Succ(Succ([3]))
[5]
```

计算 `[5] - [2]` 的过程为

``` scala
[5] - [2]
[4] - [1]
[3] - [0]
[3]
```

这样定义的数叫做*皮亚诺数(Peano numbers)*。

### 类型边界

Scala中这样写泛型的边界：

``` scala
def assertAllPos[S IntSet](r: S): S = …
```

* `S <: T`表示`S`是`T`的子类
* `S >: T`表示`S`是`T`的父类

也可以混合用：

``` scala
[S >: NonEmpty IntSet]
```

### 协变(covariance)

如果

``` scala
NonEmpty IntSet
```

那是否

``` scala
List[NonEmpty] List[IntSet]
```

也成立呢？

直觉上说是有意义的。它们的子类型关系根据类型参数变化，称为*协变*。但这对所有的类型来说都有意义吗？

先看看Java（和C#）中是怎么样的。

Java中的数组是协变的，所以有：

``` scala
NonEmpty[] 
```

但是协变数组类型会引发问题。考虑下面这段Java代码。

``` scala
NonEmpty[] a = new NonEmpty[]{new NonEmpty(1, Empty, Empty)}
IntSet[] b = a
b[0] = Empty
NonEmpty s = a[0]
```

注意最后一行中一个`Empty`被赋值给一个`NonEmpty`！

哪儿错了？

其实当执行上面这段代码时，第3行会抛出一个运行时错误`ArrayStoreException`。Java会在每个数组中保存一个类型标签，标签内容是这个数组创建时的类型，这会阻止把`Empty`赋值给`b[0]`。

这实际上把一个编译时错误变成了运行时错误，为什么Java要这样做呢？因为Java 1.5之前并不支持泛型，所以就有类似`sort(Object[] array)`这样的代码，`Object[]`使得`String[]`等类型可以被传递给`sort`。

那么什么时候应该用子类型，什么时候不应该呢？

#### [里氏替换原则(Liskov Substitution principle)](http://zh.wikipedia.org/wiki/%E9%87%8C%E6%B0%8F%E6%9B%BF%E6%8D%A2%E5%8E%9F%E5%88%99)

> If `A <: B`, then everything one can to do with a value of type `B` one should also be able to do with a value of type `A`.

（原文是

> Let q(x) be a property provable about objects x of type B. Then q(y) should be provable for objects y of type A where A 

``` scala
val a: Array[NonEmpty] = Array(new NonEmpty(1, Empty, Empty))
val b: Array[IntSet] = a
b(0) = Empty
val s: NonEmpty = a(0)
```

哪一行会报错？是第2行。

Scala 中的 `Array` 不支持协变。

