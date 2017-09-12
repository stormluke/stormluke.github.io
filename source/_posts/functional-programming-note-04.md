---
title: 函数式编程笔记 04
date: 2013-06-29
url: functional-programming-note-04
---

### 练习：找不动点

数x叫做一个函数的*不动点*，如果f(x) = x。

程序：

``` scala
object exercise {
  val tolerance = 0.0001                          //> tolerance  : Double = 1.0E-4
  def abs(x: Double) = if (x >= 0) x else -x      //> abs: (x: Double)Double
  def isCloseEnough(x: Double, y: Double) =
    abs((x - y) / x) / x //> isCloseEnough: (x: Double, y: Double)Boolean
  def fixedPoint(f: Double => Double)(firstGuess: Double) = {
    def iterate(guess: Double): Double = {
      val next = f(guess)
      if(isCloseEnough(guess, next)) next
      else iterate(next)
    }
    iterate(firstGuess)
  }                                               //> fixedPoint: (f: Double => Double)(firstGuess: Double)Double
}
```

<!-- more -->

之前的`sqrt`函数可以改为求`y = x / y`的`y`值，即`y => x / y`的不动点。

``` scala
def sqrt(x: Double) =
    fixedPoint(y => x / y)(1.0)
```

如果按上面这样写就会陷入循环，比如`sqrt(2)`时每次的`guess`在1.0、2.0反复。一个解决的办法是*平均*原序列的后继值：

``` scala
def sqrt(x: Double) =
  fixedPoint(y => (y + x / y) / 2)(1.0)
```

其实这种*求平均以稳定化*的方法很普遍，可以被抽取成一个函数。

``` scala
def averageDamp(f: Double => Double)(x: Double) = (x + f(x)) / 2
```

然后`sqrt`函数就可以这样写了：

``` scala
def sqrt(x: Double) = fixedPoint(averageDamp(y => x / y))(1.0)
```

### 类

语法什么的不想写了……

类的求值方法和函数的一样，都是call-by-value。就是说`new C(e1, …, em)`会变成`new C(v1, …, vm)`。

现在假设有这样一个类定义：

``` scala
class C(x1, …, xm) { … def f(y1, …, yn) = b …}
```

那么下面这个表达式怎么被求值？

``` scala
new C(v1, …, vm).f(w1, …, wn)
```

答案是，它会被重写成：

``` scala
[w1/y1, …, wn/yn] [v1/x1, …, vm/xm] [new C(v1, …, vm)/this] b
```

其中有三个替换过程：

1. 函数`f`的参数被求值
2. 类`C`的参数被求值
3. `this`引用被`new C(v1, …, vn)`替换

看例子：

``` scala
new Rational(1, 2).number
-> [1/x, 2/y] [] [new Rational(1, 2)/this] x
=  1
```

``` scala
new Rational(1, 2).less(new Rational(2, 3))
-> [1/x, 2/y] [new Rational(2, 3)/that] [new Rational(1, 2)/this] this.number * that.denom this.denom
=  new Rational(1, 2).number * new Rational(2, 3).denom new Rational(2, 3).number * new Rational(1, 2).denom
-> 1 * 3 2 * 2
-> true
```

### 操作符

之前如果`x`、`y`是整数，可以写`x + y`，但如果是`Rational`，则要写成`r.add(s)`。在Scala中，可以消除这种差异。

#### 中缀标记

只有一个参数的方法可以像中缀操作符那样用。

``` scala
r add s == r.add(s)
r less s == r.less(s)
r max s == r.max(s)
```

#### 宽松的标识符

操作符可以用作标识符。

标识符可以是：

* 字母组成：字母开头，后跟一系列字母或数字
* 符号组成：符号开头，后跟其他的符号
* 下划线`_`算作字母
* 字母组成的标识符也能以下划线后跟一些符号结束

标识符例子：

``` scala
x1    *    +?%&    vector_++    counter_=
```

然后就可以这样定义方法：

``` scala
def + (r: Rational) = …
def - (r: Rational) = …
```

前置标识符也是可以的，方法要以`unary_`开头：

``` scala
def unary_- : Rational = new Rational(-number, denom)
```

这样用：

``` scala
x = new Rational(1, 2)
-x
```

如果方法名以符号结尾则要把之后的`:`用空格隔开，要不就会被误认为是方法名的一部分了。

### 优先级

操作符优先级由首字母决定。

优先级的升序排列：

* 所有字母
* |
* ^
* &
* 
* = !
* :
* + -
* * / %
* 所有其他运算符

练习：给出下面的表达式的顺序

``` scala
a + b ^? c ?^ d less a ==> b | c
```

答案是：

``` scala
((a + b) ^? (c ?^ d)) less ((a ==> b) | c)
```

绝不要写这样的代码……

