---
title: 函数式编程笔记 09
date: 2013-07-05
url: functional-programming-note-09
---

### 高阶列表函数

`scaleList`：

``` scala
def scalaList(xs: List[Double], factor: Double): List[Double] = xs match {
    case Nil => xs
    case y :: ys => y * factor :: scaleList(ys, factor)
}
```

<!-- more -->

抽象成`map`：

``` scala
abstract class List[T] { …
    def map[U](f: T => U): List[U] = this match {
        case Nil => this
        case x :: xs => f(x) :: xs.map(f)
}
def scaleList(xs: List[Double], factor: Double) =
    xs map (x => x * factor)
```

`posElems`：

``` scala
def postElems(xs: List[Int]): List[Int] = xs match {
    case Nil => xs
    case y :: ys => if (y > 0) y :: posElems(ys) else posElems(ys)
}
```

抽象成`filter`：

``` scala
abstract class List[T] { …
    def filter(p: T => Boolean): List[T] = this match {
        case Nil => this
        case x :: xs => if (p(x)) x :: xs.filter else xs.filter
    }
}
def posElems(xs: List[Int]): List[Int] =
    xs filter (x => x > 0)
```

### 过滤器的变体

``` scala
xs filterNot p    // xs filter (x => !p(x))
xs partition p    // (xs filter p, xs filterNot p)
xs takeWhile p    // 满足p的最长前缀
xs dropWhile p    // 不满足p的最长前缀
xs span p         // (xs takeWhile p, xs dropWhile p)
```

练习：写一个函数`pack`，使得

``` scala
pack(List("a", "a", "a", "b", "c", "c", "a"))
```

返回

``` scala
List(List("a", "a", "a"), List("b"), List("c", "c"), List("a"))
```

答案：

``` scala
def pack[T](xs: List[T]): List[List[T]] = xs match {
    case Nil => Nil
    case x :: xs1 =>
        val (first, rest) = xs span (y => y == x)
        first :: pack(rest)
}
```

### 列表规约(reduction)

比如

``` scala
sum(List(x1, …, xn))        // = 0 + x1 + … + xn
product(List(x1, …, xn))    // = 1 * x1 * … * xn
```

`sum`函数可以这样写：

``` scala
def sum(xs: List[Int]): Int = xs match {
    case Nil => 0
    case y :: ys => y + sum(ys)
}
```

### `reduceLeft`

``` scala
List(x1, …, xn) reduceLeft op = (…(x1 op x2) op …) op xn
```

用`reduceLeft`可以简化上面两个函数：

``` scala
def sum(xs: List[Int]) = (0 :: xs) reduceLeft (_ + _)
def product(xs: List[Int]) = (1 :: xs) reduceLeft (_ * _)
```

其中每一个`_`从左向右都代表一个新参数。

### `foldLeft`

`foldLeft`和`reduceLeft`类似，但它多了一个*累加器*，当在空列表上调用`foldLeft`时就会返回它。

``` scala
(List(x1, …, xn) foldLeft z)(op) = (…(z op x1) op …) op xn
```

所以`sum`和`product`也可以这样定义：

``` scala
def sum(xs: List[Int]) = (xs foldLeft 0) (_ + _)
def product(xs: List[Int]) = (xs foldLeft 1) (_ * _)
```

`foldLeft`和`reduceLeft`可以在`List`中这样实现：

``` scala
abstract class List[T] { …
    def reduceLeft(op: (T, T) => T): T = this match {
        case Nil => throw new Error("Nil.reduceLeft")
        case x :: xs => (xs foldLeft x)(op)
    }
    def foldLeft[U](z: U)(op: (U, T) => U): U = this match {
        case Nil => z
        case x :: xs => (xs foldLeft op(z, x))(op)
    }
}
```

### `foldRight`和`reduceRight`

``` scala
List(x1, …, x{n-1}, xn) reduceRight op    // x1 op (… (x{n-1} op xn) …)
(List(x1, …, xn) foldRight acc)(op)       // x1 op (… (xn op acc) …)
```

实现：

``` scala
def reduceRight(op: (T, T) => T): T = this match {
    case Nil => throw new Error("Nil.reduceRight")
    case x :: Nil => x
    case x :: xs => op(x, xs.reduceRight(op))
}
def foldRight[U](z: U)(op: (T, U) => U): U = this match {
    case Nil => z
    case x :: xs => op(x, (xs foldRight z)(op))
}
```

练习：如果把下面函数中的`foldRight`换成`foldLeft`会发生什么错误？

``` scala
def concat[T](xs: List[T], ys: List[T]): List[T] =
    (xs foldRight ys) (_ :: _)
```

答案是类型错误。

