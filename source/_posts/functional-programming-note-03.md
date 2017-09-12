---
title: 函数式编程笔记 03
date: 2013-06-27
url: functional-programming-note-03
---


### 尾递归

来看两个函数：

``` scala
def gcd(a: Int, b: Int): Int =
    if(b == 0) a else gcd(b, a % b)

def factorial(n: Int): Int =
    if(n == 0) 1 else n * factorial(n - 1)
```

<!-- more -->

它们的调用过程分别为：

``` scala
gcd(14, 21)
if (21 == 0) 14 else gcd(21, 14 % 21)
if (false) 14 else gcd(21, 14 % 21)
gcd(21, 14 % 21)
gcd(21, 14)
if (14 == 0) 21 else gcd(14, 21 % 14)
gcd(14, 7)
gcd(7, 0)
if (0 == 0) 7 else gcd(0, 7 % 0)
7

factorial(4)
if (4 == 0) 1 else 4 * factorial(4 - 1)
4 * factorial(3)
4 * (3 * factorial(2))
4 * (3 * (2 * factorial(1)))
4 * (3 * (2 * (1 * factorial(0)))
4 * (3 * (2 * (1 * 1)))
120
```

这两个都是递归函数，区别是`gcd`在最后仅调用自己，而`factorial`在最后调用自己且乘上一个值。

如果一个函数最后一个动作是调用自己，那么它的栈帧可以被复用。这叫做*尾递归*。

> 尾递归函数是迭代过程

一般来说，如果一个函数最后的动作是调用某个函数（可能是同一个函数），则两个函数用一个栈帧就足够了。这个调用叫做*尾调用*。

在Scala中，只有最后调用自己的递归函数可以被优化。使用`@tailrec`这个注解。

``` scala
@tailrec
def gcd(a: Int, b: Int): Int = ...
```

### 尾递归版`factorial`

``` scala
object Factorial {
    def factorial(n: Int): Int = {
        def loop(acc: Int, n: Int): Int =
            if (n == 0) acc
            else loop(acc * n, n - 1)
        loop(1, n)
    }                                             //> factorial: (n: Int)Int
    factorial(4)                                  //> res0: Int = 24
}
```

### 高阶函数

如果一个函数用另一个函数做参数或返回值，则称这个函数为*高阶函数*。

比如：

``` scala
def sum(f: Int => Int, a: Int, b: Int): Int =
    if (a > b) 0
    else f(a) + sum(f, a + 1, b)
```

### 匿名函数

任何匿名函数`(x1: T1, …, xn: Tn) => E`都可以被`def`表达为：

``` scala
{ def f(x1: T1, …, xn: Tn) = E; f }
```

可以说匿名函数是*语法糖*。

### 尾递归版`sum`

``` scala
object Sum {
  def sum(f: Int => Int, a: Int, b: Int) = {
    def loop(a: Int, acc: Int): Int =
      if (a > b) acc
      else loop(a + 1, f(a) + acc)
    loop(a, 0)
  }                                               //> sum: (f: Int => Int, a: Int, b: Int)Int
  sum(x => x * x, 3, 5)                           //> res0: Int = 50
}
```

### 柯里化(Currying)

函数`sum`可以被改写成这样：

``` scala
def sum(f: Int => Int): (Int, Int) => Int = {
    def sumF(a: Int, b: Int): Int =
        if(a > b) 0
        else f(a) + sumF(a + 1, b)
    sumF
}
```

然后计算立方和的函数就可以这样写：

``` scala
sum (cube) (1, 10)
```

首先`sum(cube)`返回一个生成立方和的函数，然后对这个函数应用参数`(1, 10)`。

一般，函数参数向左结合。

``` scala
sum(cube)(1, 10) == (sum(cube))(1, 10)
```

在Scala中有个更简洁的语法，比如`sum`函数可以这样写（和上面的等价）：

``` scala
def sum(f: Int => Int)(a: Int, b: Int): Int =
    if(a > b) 0 else f(a) + sum(f)(a + 1, b)
```

通常，一个有多个参数列表的函数定义(n > 1)：

``` scala
def f(args1)…(argsn) = E
```

等价于

``` scala
def f(args1)…(args(n-1)) = {def g(argsn) = E; g}
```

其中`g`是一个临时标识。或者更简洁些：

``` scala
def f(args1)…(args(n-1)) = (argsn) => E
```

重复这个过程*n*次

``` scala
def f(args1)…(args(n-1))(argsn) = E
```

就变得和

``` scala
def f = (args1 => (args2 => …(argsn => E)…))
```

等价了。

这种定义函数和应用函数的风格叫做*柯里化*。这个名字来自于它的创造者Haskell Brooks Curry (1900-1982)，一个20世纪的逻辑学家。

事实上，这个想法可以更早地追溯到Schönfinkel和Frege，但是“柯里化”这个术语已经改不了了。

问题：下面这个函数的类型是什么？

``` scala
def sum(f: Int => Int)(a: Int, b: Int): Int = …
```

答案是

``` scala
(Int =>　Int) => (Int, Int) => Int
```

注意函数类型是向右结合的。也就是说

``` scala
Int => Int => Int
```

等价于

``` scala
Int => (Int => Int)
```

### 柯里化练习

* 写一个乘积函数`product`
* 用`product`实现一个阶乘函数`factorial`
* 写一个更一般化的函数，一般化`sum`和`product`

前两个的解：

``` scala
object exercise {
  def product(f: Int => Int)(a: Int, b:Int): Int =
    if(a > b) 1
    else f(a) * product(f)(a + 1, b)              //> product: (f: Int => Int)(a: Int, b: Int)Int
  product(x => x * x)(3, 4)                       //> res0: Int = 144
  def fact(n: Int) = product(x => x)(1, n)        //> fact: (n: Int)Int
  fact(5)                                         //> res1: Int = 120
}
```

关于第三个问题，可以发现`sum`和`product`的差别只在初始值和结合方式。把这两个不同抽取出来，就有：

``` scala
object exercise {
  def mapReduce(f: Int => Int, combine: (Int, Int) => Int, zero: Int)(a: Int, b: Int): Int =
    if(a > b) zero
    else combine(f(a), mapReduce(f, combine, zero)(a + 1, b))
                                                  //> mapReduce: (f: Int => Int, combine: (Int, Int) => Int, zero: Int)(a: Int, b:
                                                  //|  Int)Int
  def product(f: Int => Int)(a: Int, b:Int): Int = mapReduce(f, (x, y) => x * y, 1)(a, b)
                                                  //> product: (f: Int => Int)(a: Int, b: Int)Int
  product(x => x * x)(3, 4)                       //> res0: Int = 144
}
```


