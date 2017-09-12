---
title: 函数式编程笔记 07
date: 2013-07-02
url: functional-programming-note-07
---

### 变化(variance)

大体上说，允许自身元素变化的类型不应该是协变的。

设`C[T]`是一个参数化的类型，`A`、`B`类型满足`A <: B`。总体上，`C[A]`和`C[B]`间有三种可能的关系：

* `C[A] <: C[B]`：`C`是*协变*的
* `C[A] >: C[B]`：`C`是*逆变(contravariant)*的
* `C[A]`和`C[B]`互不是对方的子类型：`C`是*不变(nonvariant)*的

<!-- more -->

Scala允许通过注解类型参数来声明类型的变化：

``` scala
class C[+A] { … }    // C is covariant
class C[-A] { … }    // C is contravariant
class C[A] { … }     // C is nonvariant
```

练习：假设有两个函数类型

``` scala
type A = IntSet => NonEmpty
type B = NonEmpty => IntSet
```

那么依据里氏替换原则，它们的关系应该是什么？

答案是`A <: B`。

### 函数的类型规则

通常，函数有下面的子类型规则：

> 如果`A2 <: A1`且`B1 <: B2`，则`A1 => B1 <: A2 => B2`

所以说函数的参数类型是*逆变*的，而返回值类型是*协变*的。于是`Function1`这个trait可以修改成：

``` scala
package scala
trait Function1[-T, +U] {
    def apply(x: T): U
}
```

### 变化检查

如果把`Array`变成一个类，`update`变成一个方法，则大概会是这样：

``` scala
class Array[+T] {
    def update(x: T) …
}
```

Scala大体上会依据下面的内容检查变化注解是否满足：

* *协变*类型参数只能出现在方法的返回值中
* *逆变*类型参数只能出现在方法的参数中
* *不变*类型参数可以在任何地方出现

可以看到`Function1`是满足上面这些限制的。

练习：下面的代码有什么问题？

``` scala
trait List[+T] {
    def prepend(elem: T): List[T] = new Cons(elem, this)
}
```

问题是`prepend不能通过变化检查。（参数不能是*协变*的）

解决方法可以是：

``` scala
def prepend[U >: T](elem: U): List[U] = new Cons(elem, this)
```

练习：下面函数的返回值类型是什么？（已知`Empty <: IntSet`、`NonEmpty <: IntSet`）

``` scala
def f(xs: List[NonEmpty], x: Empty) = xs prepend x
```

答案是`List[IntSet]`。

### 分解(Decomposition)

假设现在要写一个算术表达式解释器，为了简单它只能处理数和它们的和。表达式可以用类的层次结构来表示，一个基trait `Expr`和两个子类`Number`、`Sum`。

可以用下面的方法来实现：

``` scala
trait Expr {
  def isNumber: Boolean
  def isSum: Boolean
  def numValue: Int
  def leftOp: Expr
  def rightOp: Expr
}
class Number(n: Int) extends Expr {
  def isNumber: Boolean = true
  def isSum: Boolean = false
  def numValue: Int = n
  def leftOp: Expr = throw new Error("Number.leftOp")
  def rightOp: Expr = throw new Error("Number.rightOp")
}
class Sum(e1: Expr, e2: Expr) extends Expr {
  def isNumber: Boolean = false
  def isSum: Boolean = true
  def numValue: Int = throw new Error(”Sum.numValue”)
  def leftOp: Expr = e1
  def rightOp: Expr = e2
}
def eval(e: Expr): Int = {
  if(e.isNumber) e.numValue
  else if (e.isSum) eval(e.leftOp) + eval(e.rightOp)
  else throw new Error("Unknown expression " + e)
}
```

但这有个问题，如果想新加一些表达式，比如：

``` scala
class Prod(e1: Expr, e2: Expr) extends Expr    // e1 * e2
class Var(x: String) extends Expr              // Variable 'x'
```

就需要在从基trait开始的所有类中添加区分方法(`isXXX`)和访问方法(`numValue`, `leftOp`, etc)。实际上方法的数量是平方级增长的。

### 类型测试和类型转换

要解决上面的问题，有一个不是办法的办法：使用类型测试和类型转换。

Scala允许使用类`Any`中的方法：

``` scala
def isInstanceOf[T]: Boolean    // 检查此对象是否符合`T`类型
def asInstanceOf[T]: T          // 把此对象当做`T`类型，如果失败则抛出ClassCastException
```

这和Java的类似：

``` scala
Scala                Java
x.isInstanceOf[T]    x intanceof T
x.asInstanceOf[T]    (T) x
```

使用类型测试和转换的`eval`方法这样写：

``` scala
def eval(e: Expr): Int =
    if (e.isInstanceOf[Number])
        e.asInstanceOf[Number].numValue
    else if (e.isInstanceOf[Sum])
        eval(e.asInstanceOf[Sum].leftOp) +
        eval(e.asInstanceOf[Sum].rightOp)
    else throw new Error("Unknown expression " + e)
```

这种方法的优点是不再需要区别方法，缺点是代码抽象级别低且有潜在的不安全因素。

### 面向对象分解

也可以面向相对象的方法，这样写：

``` scala
trait Expr {
    def eval: Int
}
class Number(n: Int) extends Expr {
    def eval: Int = n
}
class Sum(e1: Expr, e2: Expr) extends Expr {
    def eval: Int = e1.eval + e2.eval
}
```

但是如果现在想显示表达式的值，就要在每个子类中定义相应的新方法。并且如果想化简表达式，比如：

``` scala
a * b + a * c -> a * (b + c)
```

上面的方法是做不到的。因为这不是一个局部化简，不能被概括在单个对象的方法里。

### 模式匹配(pattern matching)

注意到：测试和访问方法的唯一目的是*逆向*类的构造过程：

* 用了哪一个子类？
* 构造器的参数是什么？

这种情况在函数式语言中很常见，包括Scala。Scala自动化了这个过程。

### Case Class

一个*case class*的定义和普通的类定义类似，除了前面有一个修饰符`case`。比如：

``` scala
trait Expr
case class Number(n: Int) extends Expr
case class Sum(e1: Expr, e2: Expr) extends Expr
```

这也隐式上定义了含有`apply`方法的同伴对象：

``` scala
object Number {
    def apply(n: Int) = new Number(n)
}
object Sum {
    def apply(e1: Expr, e2: Expr) = new Sum(e1, e2)
}
```

所以可以用`Number(1)`替代`new Number(1)`。

模式匹配是C/Java中的`switch`在类层次上的推广。在Scala中用`match`关键字。比如：

``` scala
def eval(e: Expr): Int = e match {
    case Number(n) => n
    case Sum(e1, e2) => eval(e1) + eval(e2)
}
```

### 匹配语法

* `match`后跟一系列的`case`，`pat => expr`
* 每个`case`有一个表达式`expr`和一个模式`pat`
* 如果所有模式均不匹配选择器的值，则会抛出`MatchError`异常

### 模式的格式

模式由下面的构成：

* 构造器，比如`Number`、`Sum`
* 变量，比如`e1`、`e2`
* 通配符，`_`
* 常量，比如`1`、`true`

变量始终由小写字母开头，同名变量只能在模式中出现一次。

常量由大写字母开头，除了`null`、`true`、`false`是小写。

### 求值匹配表达式

``` scala
e match { case p1 => e1 … case pn => en }
```

`e`会匹配第一个符合的模式`p`，整个匹配表达式会被重写成右边的形式，模式中的变量也会被替换为相应的部分。

* 一个构造器模式`C(p1, …, pn)`匹配所有由`p1, …, pn`参数构造的`C`类型的值
* 一个变量模式`x`匹配任何值，并且绑定到那个值
* 一个常量模式`c`匹配和`c`相等的值（`==`）

例子：

假设`eval`函数为：

``` scala
def eval(e) = e match {
    case Number(n) => n
    case Sum(e1, e2) =>
        eval(e1) + eval(e2)
}
```

则匹配过程是：

``` scala
eval(Sum(Number(1), Number(2)))
->
Sum(Number(1), Number(2)) match {
    case Number(n) => n
    case Sum(e1, e2) => eval(e1) + eval(e2)
}
->
eval(Number(1)) + eval(Number(2))
->
Number(1) match {
    case Number(n) => n
    case Sum(e1, e2) => eval(e1) + eval(e2)
} + eval(Number(2))
->
1 + eval(Number(2))
->>
3
```

把`eval`写成方法也是可以的：

``` scala
trait Expr {
    def eval: Int = this match {
        case Number(n) => n
        case Sum(e1, e2) => e1.eval + e2.eval
    }
}
```

面向对象分解和模式匹配都是不错的方法，在使用时应根据具体情境选择。如果扩展上更多的是创建新子类，则面向对象分解更适合；如果扩展上更多的是创建新方法，那么模式匹配更有优势。

