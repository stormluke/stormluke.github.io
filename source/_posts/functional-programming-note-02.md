---
title: 函数式编程笔记 02
date: 2013-06-26
url: functional-programming-note-02
---

### 求值

非基础表达式这样求值：

1. 取最左边的运算符
2. 从左向右对运算数求值
3. 对运算数施加运算符

变量名替换为定义右边的东西

<!-- more -->

### 参数和返回值

定义可以有参数和返回值：

``` scala
def square(x: Double) = x * x
def sumOfSquares(x: Double, y: Double) = square(x) + square(y)
def power(x: Double, y: Int): Double = ...
```

### 函数的求值

1. 从左向右对所有函数参数求值
2. 把函数替换为右边的东西，同时
3. 把之前的参数替换为真正的值

例如：

``` scala
sumOfSquares(3, 2+2)
sumOfSquares(3, 4)
squares(3) +　squares(4)
3 * 3 + squares(4)
9 + squares(4)
9 + 4 * 4
9 + 16
25
```

### 代换模型

上面叫做代换模型，中心思想是*把表达式退变成值*。

这个代换模型在[λ演算(lambda-calculus)](https://zh.wikipedia.org/wiki/%CE%9B%E6%BC%94%E7%AE%97)中被正式提出，是函数式编程的基础。

### 终止

每个表达式都可以在有限步内退化成一个值吗？

不是。例如：

``` scala
def loop: Int = loop
```

### 改变求值策略

先重写函数，再对参数求值。

``` scala
sumOfSquares(3, 2+2)
square(3) + square(2+2)
3 * 3 + square(2+2)
9 + square(2+2)
9 + (2+2) * (2+2)
9 + 4 * (2+2)
9 + 4 * 4
25
```

### Call-by-name和call-by-value

上面第一种叫做*call-by-value*，第二种叫做*call-by-name*。

当

* 退化表达式只包含函数
* 所有求值都能终止

时，两种都会退化成同一个值。

Call-by-value的优势是每个参数只会被求值一次。

Call-by-name的优势是可以不求没用的参数的值。

#### 终止

* 如果CBV下表达式`e`可以终止，则CBN下`e`也可以终止
* 但反过来不行

例如：

``` scala
def first(x: Int, y: Int) = x
```

Scala默认使用call-by-value。

但是也可以在函数参数前加`=>`来使用call-by-name。

这样：

``` scala
def constOne(x: Int, y: => Int) = 1
```

`constOne(1 + 2, loop)`会终止，但`constOne(loop, 1 + 2)`不会。

#### 值定义

定义也区分by-name和by-value。`def`是by-name，`val`是by-value。比如

``` scala
def x = loop
```

会终止，而

``` scala
val x = loop
```

不会。

### 例子：用牛顿法求平方根

``` scala
object Sqrt {
  def abs(x: Double) = if (x 0) -x else x       //> abs: (x: Double)Double
  def isGoodEnough(guess: Double, x: Double) =
    abs(guess * guess - x) / x 0.001            //> isGoodEnough: (guess: Double, x: Double)Boolean
  def improve(guess: Double, x: Double) =
    (guess + x / guess) / 2                       //> improve: (guess: Double, x: Double)Double
  def sqrtIter(guess: Double, x: Double): Double =
    if (isGoodEnough(guess, x)) guess
    else sqrtIter(improve(guess, x), x)           //> sqrtIter: (guess: Double, x: Double)Double
  def sqrt(x: Double) = sqrtIter(1.0, x)          //> sqrt: (x: Double)Double
  sqrt(2)                                         //> res0: Double = 1.4142156862745097
  sqrt(1e-6)                                      //> res1: Double = 0.0010000001533016628
  sqrt(1e60)                                      //> res2: Double = 1.0000788456669446E30
}
```

注意这句

``` scala
abs(guess * guess - x) / x 0.001
```

解决了数过大或过小问题。

另外Scala中递归函数必须显式指定返回类型：

``` scala
def sqrtIter(guess: Double, x: Double): Double =
```

### 区块

* 包含一系列定义或表达式
* 最后一个表达式是该区块的值
* 返回值可以被之前的定义辅助生成
* 区块也是表达式，区块可以出现在任何表达式可以出现的地方

### 可见性

* 区块内部的定义只内部可见
* 区块内部的定义会覆盖外部的同名定义

### 用区块重构

``` scala
object Sqrt {
  def abs(x: Double) = if (x 0) -x else x       //> abs: (x: Double)Double
  def sqrt(x: Double) = {
    def isGoodEnough(guess: Double) =
      abs(guess * guess - x) / x 0.001
    def improve(guess: Double) =
      (guess + x / guess) / 2
    def sqrtIter(guess: Double): Double =
      if (isGoodEnough(guess)) guess
      else sqrtIter(improve(guess))
    sqrtIter(1.0)
  }                                               //> sqrt: (x: Double)Double
  sqrt(2)                                         //> res0: Double = 1.4142156862745097
  sqrt(1e-6)                                      //> res1: Double = 0.0010000001533016628
  sqrt(1e60)                                      //> res2: Double = 1.0000788456669446E30
}
```

