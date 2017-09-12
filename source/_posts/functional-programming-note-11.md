---
title: 函数式编程笔记 11
date: 2013-07-14
url: functional-programming-note-11
---

### `Set`

`Set`是另一种集合，和序列写法相似：

``` scala
val fruit = Set("apple", "banana", "pear")
val s = (1 to 6).toSet
```

<!-- more -->

大部分序列上的操作在`Set`上也能用：

``` scala
s map (_ + 2)
fruit filter (_.startWith == "app")
s.nonEmpty
```

`Set`和序列的区别是`Set`中不包含相同元素。

### 例子：n皇后问题

``` scala
object nqueens {
  def show(queens: List[Int]) = {
    val lines =
      for (col <- queens.reverse)
      yield Vector.fill(queens.length)("* ").updated(col, "X ").mkString
    "\n" + (lines mkString "\n")
  }
  def isSafe(col: Int, queens: List[Int]): Boolean = {
    val row = queens.length
    val queensWithRow = (row - 1 to 0 by -1) zip queens
    queensWithRow forall {
      case (r, c) => col != c && math.abs(col - c) != row - r
    }
  }
  def queens(n: Int) = {
    def placeQueens(k: Int): Set[List[Int]] = {
      if (k == 0) Set(List())
      else
        for {
          queens <- placeQueens(k - 1)
          col <- 0 until n
          if isSafe(col, queens)
        } yield col :: queens
    }
    placeQueens(n)
  }
  (queens(8) take 3 map show) mkString "\n"
}
// > res0: String =
  "
  * * * * * X * *
  * * * X * * * *
  * X * * * * * *
  * * * * * * * X
  * * * * X * * *
  * * * * * * X *
  X * * * * * * *
  * * X * * * * *
  * * * * X * * *
  * * * * * * X *
  * X * * * * * *
  * * * X * * * *
  * * * * * * * X
  X * * * * * * *
  * * X * * * * *
  * * * * * X * *
  * * * * * X * *
  * * X * * * * *
  * * * * * * X *
  * * * X * * * *
  X * * * * * * *
  * * * * * * * X
  * X * * * * * *
  * * * * X * * * "
```

### `for`的翻译

`for`表达式和`map`、`flatMap`、`filter`这三个高阶函数紧密相关。这三个函数可以写成：

``` scala
def mapFun[T, U](xs: List[T], f: T => U): List[U] =
    for (x <- xs) yield f(x)
def flatMap[T, U](xs: List[T], f: T => Iterable[U]): List[U] =
    for (x <- xs; y <- f(x)) yield y
def filter[T](xs: List[T], p: T => Boolean): List[T] =
    for(x <- xs if p(x)) yield x
```

事实上，Scala编译器将`for`表达式翻译成`map`、`flatMap`和懒求值的`filter`变体。

Scala编译器用下面的方式翻译：

#### 1. 简单的`for`表达式

``` scala
for (x <- e1) yield e2
```

被翻译成

``` scala
e1.map(x => e2)
```

#### 2. 带过滤器的`for`表达式

``` scala
for (x <- e1 if f; s) yield e2
```


其中`f`是过滤器，`s`是剩下的生成器过滤器序列。这会被翻译成：

``` scala
for (x <- e1.withFilter(x => f); s) yield e2
```

可以认为`withFilter`是不生成临时中间列表的`filter`。

#### 3. 多个生成器的`for`表达式

``` scala
for (x <- e1; y <- e2; s) yield e3
```

其中`s`是剩下的生成器过滤器序列，这会被翻译成：

``` scala
e1.flatMap(x => for (y <- e2; s) yield e3)
```

可以看到上面三种情况概括了所有的`for`表达式形式。

例子：

``` scala
for {
    i <- 1 until n
    j <- 1 until i
    if isPrime(i + j)
} yield (i, j)
```

用上面的方法会被翻译为：

``` scala
(1 until n).flatMap =>
    (1 until i).withFilter(j => isPrime(i + j)).map(j => (i, j))
```

### `for`的推广

有趣的是，`for`不仅仅只能应用在列表或序列上，它只要求`map`、`flatMap`、`withFilter`这三个方法存在即可。

所以可以将其应用在数组、遍历器、数据库、XML数据、可选值、分析器等等中。

`for`是Scala数据库框架ScalaQuery、Slick的基础。微软的LINQ也有同样的思想。

### `Map`

`Map[Key, Value]`是一种将键`Key`和值`Value`结合起来的数据结构。

例如：

``` scala
val romanNumbers = Map("I" -> 1, "V" -> 5, "X" -> 10)
val capitalOfCountry = Map("US" -> "Washington", "Switzerland" -> "Bern")
```

类`Map[Key, Value]`继承自`Iterable[(Key, Value)]`，因此大部分`Iterable`的方法能在`Map`上用：

``` scala
val countryOfCapital = capitalOfCounty map {
    case(x, y) => (y, x)
}
```

事实上，`->`语法就相当于写二元组`(key, value)`。

类`Map[Key, Value]`也继承了函数类型`Key => Value`，所以`Map`可以像函数一样使用：

``` scala
capitalOfCountry("Andorra")    // java.util.NoSuchElementException: key not found: Andorra
``` scala

想要在不知道是否包含键的情况下访问`Map`，可以使用`get`操作：

``` scala
capitalOfCountry get "US"         // Some("Washington")
capitalOfCountry get "Andorra"    // None
```

`get`的返回值是一个`Option`值。

### `Option`

`Option`是这样定义的：

``` scala
trait Option[+A]
case class Some[+A](value A) extends Option[A]
object None extends Option[Nothing]
```

`map get key`表达式返回：

* `None`：如果`map`不包含键`key`
* `Some(x)`：如果`map`包含键`key`

因为`Option`被定义为case类，所以用模式匹配分解它：

``` scala
def showCapital(country: String) = capitalOfCountry.get(country) match {
    case Some(capital) => capital
    case None => "missing data"
}
showCapital("US")         // "Washington"
showCapital("Andorra")    // "missing data"
```

### `sorted`和`groupBy`

除了`for`表达式外另两个有用的SQL查询是`groupBy`和`orderBy`。

`orderBy`在集合上可以用`sortWith`和`sorted`来表达：

``` scala
val fruit = List("apple", "pear", "orange", "pineapple")
fruit sortWith (_.length // List("pear", "apple", "orange", "pineapple")
fruit.sorted                            // List("apple", "orange", "pear", "pineapple")
```

`groupBy`在Scala集合上也能用，它通过*鉴别器函数*`f`将一个集合分解成一个包含多个集合的map。

``` scala
fruit groupBy (_.head)    // > Map(p -> List(pear, pineapple),
                          // |     a -> List(apple),
                          // |     o -> List(orange))
```

### 例子：多项式

假设`x^3 - 2x + 5`表示成`Map(0 -> 5, 1 -> -2, 3 -> 1)`，则可以设计一个类`Poly`来表示多项式。

到目前为止，map还只是*部分函数*，如果键不存在，`map(key)`就会抛出异常。可以用`withDefaultValue`操作将map转变为完全函数：

``` scala
val cap1 = capitalOfCountry withDefaultValue ""
cap1("Andorra")    // ""
```

多项式可以先写成这样：

``` scala
object polynomials {
  class Poly(terms0: Map[Int, Double]) {
    val terms = terms0 withDefaultValue 0.0
    def +(other: Poly) = new Poly(terms ++ (other.terms map adjust))
    def adjust(term: (Int, Double)): (Int, Double) = {
      val (exp, coeff) = term
      terms get exp match {
        case Some(coeff1) => exp -> (coeff + coeff1)
        case None => exp -> coeff
      }
    }
    override def toString =
      (for ((exp, coeff) <- terms.toList.sorted.reverse) yield coeff + "x^" + exp) mkString " + "
  }
  val p1 = new Poly(Map(1 -> 2.0, 3 -> 4.0, 5 -> 6.2))    // > p1: Poly = 6.2x^5 + 4.0x^3 + 2.0x^1
  val p2 = new Poly(Map(0 -> 3.0, 3 -> 7.0))              // > p2: Poly = 7.0x^3 + 3.0x^0
  p1 + p2                                                 // > res0: Poly = 6.2x^5 + 11.0x^3 + 2.0x^1 + 3.0x^0
}
```

但是每次都要用`Map`来包裹参数很不方便。可以用*多重参数*来改进它：

``` scala
def this(bindings: (Int, Double)*) = this(bindings.toMap)
```

其中`bindings`的类型为`Seq[(Int, Double)]`。

加法操作也可以用`foldLeft`来改写：

``` scala
class Poly(terms0: Map[Int, Double]) {
    def this(bindings: (Int, Double)*) = this(bindings.toMap)
    val terms = terms0 withDefaultValue 0.0
    def +(other: Poly) = new Poly((other.terms foldLeft terms)(addTerm))
    def addTerm(terms: Map[Int, Double], term: (Int, Double)): Map[Int, Double] = {
        val (exp, coeff) = term
        terms + (exp -> (coeff + terms(exp)))
    }
    override def toString =
        (for ((exp, coeff) <- terms.toList.sorted.reverse) yield coeff + "x^" + exp) mkString " + "
}
```

实际上`foldLeft`版本比`++`版本的更有效率，原因是`foldLeft`直接将系数加到`terms`中得到了新的map，而`++`版本中创建了一个临时map后再将他们连接起来。

