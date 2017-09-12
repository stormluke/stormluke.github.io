---
title: 响应式编程笔记 03
date: 2014-06-07
url: reactive-programming-note-03
---


### 函数和状态

到目前为止，所见到的程序都是无副作用的。所以，*时间*的概念并不重要。对所有终止的程序来说，任何动作顺序都会返回一样的结果。这在替换模型中也有所体现。

### 回忆：替换模型

程序可以用*重写 (rewriting)* 的方法来求值。函数调用中最重要的重写规则是：

``` scala
def f(x1, ..., xn) = B; ... f(v1, ..., vn)
->
def f(x1, ..., xn) = B; ... [v1/x1, ..., vn/xn] B
```

<!-- more -->

若有下面两个函数 `iterate` 和 `square`：

``` scala
def iterate(n: Int, f: Int => Int, x: Int) =
  if (n == 0) x else iterate(n - 1, f, f(x))
def square(x: Int) = x * x
```

那么调用 `iterate(1, square, 3)` 会被重写为：

``` scala
-> if (1 == 0) 3 else iterate(1 - 1, square, square(3))
-> iterate(0, square, square(3))
-> iterate(0, square, 3 * 3)
-> iterate(0, square, 9)
-> if (0 == 0) 9 else iterate(0 - 1, square, square(9))
-> 9
```

重写可以在一项的任意位置进行，并且所有由重写生成的结果都一样。这是函数式编程背后 [λ 演算](http://stormluke.me/reactive-programming-note-03/http://zh.wikipedia.org/wiki/%CE%9B%E6%BC%94%E7%AE%97)理论的一个重要结论。也叫做 [Church-Rosser Theroem](http://stormluke.me/reactive-programming-note-03/http://en.wikipedia.org/wiki/Church%E2%80%93Rosser_theorem)。

比如

``` scala
if (1 == 0) 3 else iterate(1 - 1, square, square(3))
```

可以被重写为：

``` scala
iterate(0, square, square(3))
```

也可以被重写为：

``` scala
if (1 == 0) 3 else iterate(1 - 1, square, 3 * 3)
```

不管怎样它们最终的结果都是 `9`。

### 有状态的对象

一般会把世界描绘成一系列的对象，它们中的一些的状态会随着时间而*改变*。称一个对象*有状态*是指它的行为受到其历史的影响。

比如一个银行账号是有状态的，因为它对“能取100元吗”这个问题的回答在此账号的生命周期中是有可能变化的。

所有形式的可变状态都是从变量构成的。变量的定义的写法和值定义类似，但是将 `val` 关键字换成 `var`：

``` scala
var x: String = "abc"
var count = 111
```

像值定义那样，变量定义将一个值和一个名称联系起来。但不同的是，这个联系可以通过*赋值 (assignment)* 操作改变，就像 Java 中那样：

``` scala
x = "hi"
count = count + 1
```

在实践中，有状态的对象通常被表现成包含一些变量成员的对象。比如银行账号类：

``` scala
class BankAccount {
  private var balance = 0
  def deposit(amount: Int): Unit = {
    if (amount > 0) balance = balance + amount
  }
  def withdraw(amount: Int): Int =
    if (0 
      balance = balance - amount
      balance
    } else throw new Error("insufficient funds")
}
```

类 `BankAccount` 定义了一个变量 `balance` 来表示当前的账户余额，方法 `deposit` 和 `withdraw` 通过赋值来改变 `balance` 的值。很明显账户是有状态的对象。

回一下函数式编程第七周中对流（懒求值的序列）的定义，可以不用 `lazy val`，而用一个变量来实现非空的流：

``` scala
def cons[T](hd: T, tl: => Stream[T]) = new Stream[T] {
  def head = hd
  private var tlOpt: Option[Stream[T]] = None
  def tail: T = tlOpt match {
    case Some(x) => x
    case None => tlOpt = Some(tl); tail
  }
}
```

现在的问题是，`cons` 的返回值是有状态的对象吗？

答案是两种都有可能。如果 `tl` 是有状态的对象，则返回值也是有状态的，如果 `tl` 是无状态的，则返回值一样无状态。

再看下面这个类：

``` scala
class BankAccountProxy(ba: BankAccount) {
  def deposit(amount: Int): Unit = ba.deposit(amount)
  def withdraw(amount: Int): Int = ba.withdraw(amount)
}
```

它的成员中并不包含变量，那它的实例有状态吗？显然是有的，因为它和 `BankAccount` 的行为一致，即依赖于历史状态。


