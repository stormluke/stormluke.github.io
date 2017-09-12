---
title: 响应式编程笔记 04
date: 2014-06-12
url: reactive-programming-note-04
---

### 同一性和变化

赋值给辨别两个表达式是否*相同*带了新问题。当不考虑赋值时，可以这样写：

``` scala
val x = E; val y = E
```

<!-- more -->

其中 `E` 为任意的表达式，可以合理假定 `x` 和 `y` 是一样的，就是说也可以这样写：

``` scala
val x = E; val y = x
```

这个特性一般称为[*引用透明（referential transparency）*](http://en.wikipedia.org/wiki/Referential_transparency_%28computer_science%29)。

但当引入赋值后，这两个式子就不同了，比如：

``` scala
val x = new BankAccount
val y = new BankAccount
```

“相同”这个词的准确意义由[*运作相等（operational equivalence）*](https://en.wikipedia.org/wiki/Operational_equivalence)这个特性定义，非形式化来说就是假设有两个定义 `x` 和 `y`，若没有一个*可行的测试*能区分它们，则称 `x` 和 `y` 是运作相等的。

要测试 `x` 和 `y` 是否相同，应该：

* 对其执行涉及 `x` 和 `y` 的任意操作序列 `S`，观察可能的结果

``` scala
val x = new BankAccount
val y = new BankAccount
f(x, y)
```

* 然后执行另外一个操作序列 `S'`，`S'` 是将 `S` 中的 `y` 全部换成 `x` 后得到的序列

``` scala
val x = new BankAccount
val y = new BankAccount
f(x, x)
```

* 如果这两个结果不同，则 `x` 和 `y` 肯定不同
* 如果所有可能的序对 `(S, S')` 的结果都相同，则 `x` 和 `y` 相同

基于这个定义，来看看下面的表达式是否相同：

``` scala
val x = new BankAccount
val y = new BankAccount
```

用这个测试序列来执行：

``` scala
x deposit 30    // val res1: Int = 30
y withdraw 20   // java.lang.Error:insufficient funds
```

现在将序列中的 `y` 全部换为 `x`，得到：

``` scala
x deposit 30    // val res1: Int = 30
x withdraw 20   // val res2: Int = 10
```

可以看到两个结果不一致，所以 `x` 和 `y` 是不同的。

另一方面，若定义：

``` scala
val x = new BankAccount
val y = x
```

没有任何操作序列可以区分它们，所以 `x` 和 `y` 在这个例子中是相同的。

上面的例子展示了在引入赋值时替换模型变得不正确。可以通过引入一个 *store* 来继续适应替换模型，但这会变得相当复杂。

### 循环

命题：变量足以建模所有命令式程序。

但是像循环之类的控制语句怎么办？其实可以用函数来实现它们。

比如说，`while` 循环语句可以用函数 `WHILE` 来定义：

``` scala
def WHILE(condition: => Boolean)(command: => Unit): Unit =
  if (condition) {
    command
    WHILE(condition)(command)
  }
  else ()
```

有两点值得注意：一是 `condition` 和 `command` 必须使用名传递（passed by name），这样它们才会在每个迭代中被重新求值；二是 `WHILE` 是尾递归的，所以它可以在常量栈空间内执行。

练习：写一个实现 `repeat` 循环的函数：

``` scala
REPEAT {
  command
} (condition)
```

答案：

``` scala
def REPEAT(command: => Unit)(condition: => Boolean) = {
  command
  if (condition) ()
  else REPEAT(command)(condition)
}
```

练习：用函数实现 `repeat-until`：

``` scala
REPEAT {
  command
} UNTIL (condition)
```

一个可行的方法是（来自[Ilya Golberg](https://class.coursera.org/reactive-001/forum/thread?thread_id=355#post-1545)，混合了 `repeat-until` 和 `repeat`）：

``` scala
def REPEAT(command: => Until) = new {
  /* REPEAT {command} UNTIL (condition) */
  def UNTIL(condition: => Boolean) {
    command
    if (!condition) UNTIL(condition)
    else ()
  }
  /* REPEAT {command} (condition) */
  def apply(condition: => Boolean) = UNTIL(condition)
}
```

Java 中的典型 `for` 循环是不能由高阶函数建模的，比如下面这个 Java 程序：

``` scala
for (int i = 1; i 3; i = i + 1) { System.out.print(i + " "); }
```

原因是 `for` 的参数中包含了变量 `i` 的*声明*，这个变量在其他参数和循环体中都是可见的。

但其实在 Scala 中有一个和 Java 扩展 `for` 循环类似的 `for` 循环：

``` scala
for (i 1 until 3) { System.out.print(i + " ") }
```

`for` 循环和 `for` 表达式的翻译过程类似，不过用 `foreach` 组合子（combinator）替换了 `map` 和 `flatMap`。`foreach` 在包含元素类型 `T` 的集合中如下定义：

``` scala
def foreach(f: T => Unit): Unit =
  // apply f to each element of the collection
```

比如，下面的式子：

``` scala
for (i 1 until 3; j "abc") println(i + " " + j)
```

会被翻译为：

``` scala
(1 until 3) foreach (i => "abc" foreach (j => println(i + " " + j)))
```


