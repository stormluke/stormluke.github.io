---
title: 响应式编程笔记 02
date: 2014-05-17
url: reactive-programming-note-02
---

### 回顾：Case 类

Case 类是 Scala 处理复杂数据的理想方式。

比如表示 JSON 类型的数据，可以这样：

``` scala
abstract class JSON
case class JSeq(elems: List[JSON]) extends JSON
case class JObj(bindings: Map[String, JSON]) extends JSON
case class JNum(num: Double) extends JSON
case class JStr(str: String) extends JSON
case class JBool(b: Boolean) extends JSON
case object JNull extends JSON
```
<!-- more -->

### 模式匹配

这是将 JSON 数据转换成字符串的方法：

``` scala
def show(json: JSON): String = json match {
  case JSeq(elems) =>
    "[" + (elems map show mkString ", ") + "]"
  case JObj(bindings) =>
    val assocs = bindings map {
      case (key, value) => "\"" + key + "\": " + show(value)
    }
    "{" + (assocs mkString ", ") + "}"
  case JNum(num) => num.toString
  case JStr(str) => "\"" + str + "\""
  case JBool(b) => b.toString
  case JNull => "null"
}
```

### 函数是对象

问题：下面表达式的类型是什么？

``` scala
{ case (key, value) => key + ": " + value }
```

当它单独出现时，它是没有类型的，需要给它规定一个期望的类型。

上面代码中 `map` 方法期望的类型是 `JBinding => String`，其中 `JBinding` 为：

``` scala
type JBinding = (String, JSON)
```

在 Scala 中，所有实体类型都是某个类或 trait，函数也不例外。

``` scala
JBinding => String
```

其实是

``` scala
scala.Function1[JBinding, String]
```

的缩写形式，`scala.Function1`是一个 trait，`JBinding` 和 `String` 则是它的类型参数。

### Function1 Trait

这是`trait Function1`的大概样子：

``` scala
trait Function1[-A, +R] {
  def apply(x: A): R
}
```

之前的模式匹配块会被扩展为`Function1`的一个实例：

``` scala
new Function1[JBinding, String] {
  def apply(x: Binding) = x match {
    case (key, value) => key + ": " + show(value)
  }
}
```

### Function 的子类

函数是 trait 的一个好处是可以用子类来扩展它。

比如 map 是从键到值的函数：

``` scala
trait Map[Key, Value] extends (Key => Value)
```

序列是从 `Int` 到值的函数：

``` scala
trait Seq[Elem] extends Int => Elem
```

这就是 `elems(i)` 可以这样写的原因。

### 不完全函数 (Partial Function)

有如下定义的函数：

``` scala
val f: String => String = { case "ping" => "pong" }
```

它的类型为 `String => String`，但是这个函数没有在整个域（所有字符串）上定义。

``` scala
f("pong") // 会抛出 MatchError 异常
```

有在运行一个函数之前就能测试它能接受哪些参数的方法吗？确实有：

``` scala
val f: PartialFunction[String, String] = { case "ping" => "pong" }
f.isDefinedAt("ping") // true
f.isDefinedAt("pong") // false
```

不完全函数是这样定义的：

``` scala
trait PartialFunction[-A, +R] extends Function1[-A, +R] {
  def apply(x: A): R
  def isDefinedAt(x: A): Boolean
}
```

如果期望类型是 `PartialFunction`，Scala 编译器会将

``` scala
{ case "ping" => "pong" }
```

扩展为：

``` scala
new PartialFunction[String, String] {
  def apply(x: String) = x match {
    case "ping" => "pong"
  }
  def isDefinedAt(x: String) = x match {
    case "ping" => true
    case _ => false
  }
}
```

### 练习

这个函数：

``` scala
val f: PartialFunction[List[Int], String] = {
  case Nil => "one"
  case x :: rest =>
    rest match {
      case Nil => "two"
    }
}
```

`g.isDefinedAt(List(1, 2, 3))` 的返回值是？

答案为 `true`。但应注意尽管测试为 `true`，但当以 `List(1, 2, 3)` 为参数调用 `f` 时会发生错误。也就是说 `isDefinedAt` 只保证最外层的模式匹配正确。

### 回顾：集合

Scala 包含丰富的集合类层次。所有集合类型共享一些通用方法。

核心方法：

* `map`
* `flatMap`
* `filter`

另外还有：

* `foldLeft`
* `foldRight`

各方法在列表 (List) 上的理想化实现：

``` scala
abstract class List[+T] {
  def map[U](f: T => U): List[U] = this match {
    case x :: xs => f(x) :: xs.map(f)
    case Nil => Nil
  }
  def flatMap[U](f: T => List[U]): List[U] = this match {
    case x :: xs => f(x) ++ xs.flatMap(f)
    case Nil => Nil
  }
  def filter(p: T => Boolean): List[T] = this match {
    x :: xs =>
      if (p(x)) x :: xs.filter(p) else xs.filter(p)
    case Nil => Nil
  }
}
```

在实践中，这些方法的具体实现和类型并不和上面相同，这样做是想:

* 让它们可以应用在不同的集合上面，不仅是列表
* 让它们在列表上是尾递归的

### For 表达式

For 表达式可以简化 `map`、`flatMap`、`filter` 的组合。

``` scala
(1 until n) flatMap (i =>
  (1 until i) filter (j => isPrime(i + j)) map
    (j => (i, j)))
```

可以写成：

``` scala
for {
  i <- 1 until n
  j <- 1 until i
  if isPrime(i + j)
} yield (i, j)
```

### For 的翻译

Scala 编译器会把 for 表达式翻译成 `map`、`flatMap` 和 `filter` 的一种懒求值变体的组合。

* 简单的 for 表达式

``` scala
for (x <- e1) yield e2
```

会被翻译为：

``` scala
e1.map(x => e2)
```

* 包含过滤器的 for 表达式

``` scala
for (x <- e1 if f; s) yield e2
```

其中 `f` 是过滤器，`s` 是（可能为空）一系列的生成器和过滤器。它会被翻译为：

``` scala
for (x <- e1.withFilter(x => f); s) yield e2
```

可以认为 `withFilter` 是 `filter` 的一种变体，它不产生临时列表，而是直接过滤 `map` 或 `flatMap`中剩余的元素。

* 包含多个生成器的 for 表达式

``` scala
for (x <- e1; y <- e2; s) yield e3
```

会被翻译为：

``` scala
e1.flatMap(x => for (y <- e2; s) yield e3)
```

然后继续递归翻译。

### For 表达式和模式匹配

生成器的左侧也可以是模式。

比如：

``` scala
val data: List[JSON] = ...
for {
  JObj(bindings) <- data
  JSeq(phones) = bindings("phoneNumbers")
  JObj(phone) <- phones
  JStr(digits) = phone("number")
  if (digits) startsWith "212"
} yield (bindings("firstName"), bindings("lastName"))
```

如果 `pat` 是有一个变量 `x` 的模式，将会把

``` scala
pat <- expr 
```

翻译为：

``` scala
x <- expr withFilter {
       case pat => true
       caes _ => false
     } map {
       case pat => x
     }
```

### 练习

``` scala
for {
  x <- 2 to N
  y <- 2 to x
  if (x % y == 0)
} yield (x, y)
```

会被翻译为？

答案：

``` scala
(2 to N) flatMap (x =>
  (2 to x) withFilter (y =>
    x % y == 0) map (y => (x, y)))
```

### For 表达式的其他用法：函数式随机生成器

For 可以应用在任何实现了 `map`、`flatMap`、`withFilter` 的对象上。

现在以一个能生成任意类型的随机生成器为例。

首先定义 trait：

``` scala
trait Generator[+T] {
  def generate: T
}
```

一些实例：

``` scala
val integers = new Generator[Int] {
  val rand = new java.util.Random
  def generate = rand.nextInt()
}
val booleans = new Generator[Boolean] {
  def generate = integers.generate > 0
}
val pairs = new Generator[(Int, Int)] {
  def generate = (integers.generate, integers.generate)
}
```

会发现有许多 `new Generator` 重复，能避免吗？理想中的写法是：

``` scala
val booleans = for (x <- integers) yield x > 0
def pairs[T, U](t: Generator[T], u: Generator[U]) = for {
  x <- t
  y <- u
} yield (x, y)
```

这其实会生成：

``` scala
val booleans = integers map (x => x > 0)
def pairs[T, U](t: Generator[T], u: Generator[U]) =
  t flatMap (x => u map (y => (x, y)))
```

所以说要定义 `map` 和 `flatMap`。下面是一个更便捷的 `Generator` 版本：

``` scala
trait Generator[+T] {
  self =>
  def generate: T
  def map[S](f: T => S): Generator[S] = new Generator[S] {
    def generate = f(self.generate)
  }
  def flatMap[S](f: T => Generator[S]): Generator[S] = new Generaotr[S] {
    def generate = f(self.generate).generate
  }
}
```

那么上面的 `booleans` 和 `pairs` 展开过程为：

``` scala
val booleans = new Generator[Boolean] {
  def generate = (x: Int => x > 0)(integers.generate)
}
val booleans = new Generator[Boolean] {
  def generate = integers.generate > 0
}
```

``` scala
def pairs[T, U](t: Generator[T], u: Generator[U]) = t flatMap {
  x => new Generator[(T, U)] { def generate = (x, u.generate) }
}
def pairs[T, U](t: Generator[T], u: Generator[U]) = new Generator[(T, U)] {
  def generate = (new Generator[(T, U)] {
    def generate = (t.generate, u.generate)
  }).generate
}
def pairs[T, U](t: Generator[T], u: Generator[U]) = new Generator[(T, U)] {
  def generate = (t.generate, u.generate)
}
```

然后可以定义一些实用方法：

``` scala
def single[T](x: T): Generator[T] = new Generator[T] {
  def generate = x
}
def choose(lo: Int, hi: Int): Generator[Int] =
  for (x <- integers) yield lo + x % (hi - lo)
def oneOf[T](xs: T*): Generator[T] =
  for (idx <- choose(0, xs.length)) yield xs(idx)
```


现在就可以定义一个生成列表的随机生成器了：

``` scala
def lists: Generator[List[Int]] = for {
  isEmpty <- booleans
  list <- if (isEmpty) emptyLists else nonEmptyLists
} yield list
def emptyLists = single(Nil)
def nonEmptyLists = for {
  head <- integers
  tail <- lists
} yield head :: tail
```

也可以定义一个生成二叉树的随机生成器：

``` scala
trait Tree
case class Inner(left: Tree, right: Tree) extends Tree
case class Leaf(x: Int) extends Tree
def leafs: Generator[Leaf] = for {
  x <- integers
} yield Leaf(x)
def inners: Generator[Inner] = for {
  l <- trees
  r <- trees
} yield Inner(l, r)
def trees: Generator[Tree] = for {
  isLeaf <- booleans
  tree <- if (isLeaf) leafs else inners
} yield tree
```

### 随机生成器的一个应用：随机测试 (Random Testing)

关于单元测试：

* 包含一些测试输入和一个后置条件 (postcondition)
* 后置条件是期望结果的一个属性
* 验证此程序是否满足后置条件

若将测试输入替换为随机生成的输入，就是随机测试。

用随机生成器可以写一个测试函数：

``` scala
def test[T](g: Generator[T], numTimes: Int = 100)(test: T => Boolean): Unit = {
  for (i <- 0 until numTimes) {
    val value = g.generate
    assert(test(value), "test failed for " + value)
  }
  println("passed " + numTimes + " tests")
}
```

如果有下面的一个使用例子：

``` scala
test(pairs(lists, lists)) {
  case (xs, ys) => (xs ++ ys).length > xs.length
}
```

它会始终通过测试吗？

答案是不会。因为 `lists` 随机生成器会生成空列表。

关于 ScalaCheck：一个测试工具，使用它写测试时只用写待测属性而不用写具体测试过程。比如这样：

``` scala
forAll { (l1: List[Int], l2: List[Int]) =>
  l1.size + l2.size == (l1 ++ l2).size
}
```

### 单子 (Monad)

可以看出包含 `map` 和 `flatMap` 的数据结构很普遍。实际上有个专门的名称来表述这些满足一定代数条件的数据结构类别，这就是*单子*。

一个单子 `M` 是一个拥有 `flatMap` 和 `unit` 这两种操作的参数化类型 `M[T]`，它必须满足一些条件。

``` scala
trait M[T] {
  def flatMap[U](f: T => M[U]): M[U]
}
def unit[T](x: T): M[T]
```

在其他文献中，`flatMap` 也常被称为 `bind`。

* `List` 是单子，其中 `unit(x) = List(x)`
* `Set` 是单子，其中 `unit(x) = Set(x)`
* `Option` 是单子，其中 `unit(x) = Some(x)`
* `Generator` 是单子，其中 `unit(x) = single(x)`

在 Scala 中，`flatMap` 是这些类型上的一个操作，而 `unit` 对于每个单子都不相同。

对每个单子， `map` 可以被定义为 `flatMap` 和 `unit` 的组合：

``` scala
m map f == m flatMap (x => unit(f(x)))
        == m flatMap (f andThen unit)
```

### 单子的条件

一种类型若为合格的单子需要满足三个条件：

* 结合律（associativity）

``` scala
m flatMap f flatMap g == m flatMap (x => f(x) flatMap g)
```

* 左单位元（left unit）

``` scala
unit(x) flatMap f == f(x)
```

* 右单位元（right unit）

``` scala
m flatMap unit == m
```

### [](http://stormluke.me/reactive-programming-note-02/%23%E4%BE%8B%E5%AD%90 "例子")例子

现在以 `Option` 为例来检验这三个条件。`Option` 的 `flatMap` 为：

``` scala
abstract class Option[+T] {
  def flatMap[U](f: T => Option[U]): Option[U] = this match {
    case Some(x) => f(x)
    case None => None
  }
}
```

下面来验证 `Some(x) flatMap f == f(x)` (左单位元)：

``` scala
    Some(x) flatMap f
==  Some(x) match {
      case Some(x) => f(x)
      case None => None
    }
==  f(x)
```

再来验证 `opt flatMap Some == opt` (右单位元)：

``` scala
    opt flatMap Some
==  opt match {
      case Some(x) => Some(x)
      case None => None
    }
==  opt
```

最后验证 `opt flatMap f flatMap g == opt flatMap (x => f(x) flatMap g)` (结合律)：

``` scala
    opt flatMap f flatMap g
==  opt match { case Some(x) => f(x) case None => None }
        match { case Some(y) => g(y) case None => None }
==  opt match {
      case Some(x) =>
        f(x) match { case Some(y) => g(y) case None => None }
      case None =>
        None match { case Some(y) => g(y) case None => None }
    }
==  opt match {
      case Some(x) =>
        f(x) match { case Some(y) => g(y) case None => None }
      case None => None
    }
==  opt match {
      case Some(x) => f(x) flatMap g
      case None => None
    }
==  opt flatMap (x => f(x) flatMap g)
```

### 三条件对 for 表达式的意义

可以看到单子类型的表达式一贯被写为 for 表达式，那么对它来说这三个条件的意义是什么呢？

* 结合律本质上是说可以“内联 (inline)”嵌套 for 表达式：

``` scala
    for (y <- for (x <- m; y <- f(x)) yield y
         z <- g(y)) yield z
==  for (x <- m;
         y <- f(x)
         z <- g(y)) yield z
```

* 右单位元是指：

``` scala
    for (x <- m) yield x
==  m
```

* 左单位元在 for 表达式中没有对应

### 另一个类型：Try

`Try` 类似于 `Option`，不同之处是将 `Some` / `None` 替换成了包含一个值的 `Success` 和包含一个异常的`Failure`：

``` scala
abstract class Try[+T]
case class Success[T](x: T) extends Try[T]
case class Failure(ex: Exception) extends Try[Nothing]
```

`Try` 被用来在线程或计算机间传递计算结果，这些计算可能失败并包含异常。

`Try` 可以包裹任意的计算：

``` scala
Try(expr) // 返回 Success(someValue) 或 Failure(someException)
```

下面是 `Try` 的一个实现：

``` scala
object Try {
  def apply[T](expr: => T): Try[T] =
    try Success(expr)
    catch {
      case NonFatal(ex) => Failure(ex)
    }
}
```

像 `Option` 一样，`Try` 类型的计算可以由 for 表达式构成：

``` scala
for {
  x <- computeX
  y <- computeY
} yield f(x, y)
```

如果 `computeX` 和 `computeY` 均成功且结果是 `Success(x)` 和 `Success(y)`，那么会返回 `Success(f(x, y))`。如果其中一个失败并有一个异常 `ex`，则会返回 `Failure(ex)`。

下面是 `Try` 中 `flatMap` 和 `map` 的定义：

``` scala
abstract class Try[T] {
  def flatMap[U](f: T => Try[U]): Try[U] = this match {
    case Success(x) => try f(x) catch { case NonFatal(ex) => Failure(ex) }
    case fail: Failure => fail
  }
  def map[U](f: T => U): Try[U] = this match {
    case Success(x) => Try(f(x))
    case fail: Failure => fail
  }
}
```

所以对一个 `Try` 类型值 `t` 有：

``` scala
t map f == t flatMap (x => Try(f(x)))
        == t flatMap (f andThen Try)
```

### 练习

看起来 `Try` 像是一个单子，其中 `unit = Try`，对吗？

其实它不满足左单位元：

``` scala
Try(expr) flatMap f != f(expr)
```

实际上左侧永远不会产生非致命异常，但右侧会产生由 `expr` 或 `f` 抛出的任何异常。

因此，`Try` 以一个单子条件的代价换来了另外一个特性，这在下面的情景中更有用：

> 由 `Try`、`map`、`flatMap` 构成的表达式永远不会抛出非致命异常

称这个为“防弹”原则 (“bullet-proof” principle)。

### 小结

从上面可以看出 for 表达式不仅仅对于集合有用。很多其他类型也定义了 `map`、`flatMap`、`withFilter` 操作和由此而来的 for 表达式。

很多定义了 `flatMap` 的类型都是单子。如果它们也定义了 `withFilter`，则可以称为“包含0的单子” (monads with zero)。

单子的三个条件给库的API设计提供了有用的指导。

