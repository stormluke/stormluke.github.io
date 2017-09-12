---
title: 函数式编程笔记 10
date: 2013-07-13
url: functional-programming-note-10
---


### `Vector`

列表是*线性*的，而向量是树形的，所以向量比列表有更平衡的访问时间(log32(n))。

向量和列表的定义方法类似：

``` scala
val nums = Vector(1, 2, 3, -88)
val people = Vector("Bob", "James", "Peter")
```

<!-- more -->

基本的运算符：

``` scala
x +: xs    // 创建一个首元素是x，后跟xs所有元素的向量
xs :+ x    // 相反
```

### 集合的继承层次

```
Iterable <-| Map
           | Set
           | Seq <-| List
                   | Vector
                   | Range
                   | String
                   | Array
```

其中`Array`和`String`支持和`Seq`相同的操作，并且可以隐式转换成`Seq`，但它们并不是`Seq`的子类，原因是它们来自于Java。

### `Range`

`Range`是整数的序列，三个运算符：

``` scala
val r: Range = 1 until 5    // 1, 2, 3, 4
val s: Range = 1 to 5       // 1, 2, 3, 4, 5
1 to 10 by 3                // 1, 4, 7, 10
6 to 1 by -2                // 6, 4, 2
```

### 更多的`Seq`操作符

``` scala
xs exists p     // 是否包含满足p条件的元素
xs forall p     // 是否所有元素都满足p条件
xs zip ys       // 返回包含二元组的列表，其中元素是xs和ys
xs unzip        // zip的逆操作，返回包含列表的二元组
xs.flatMap f    // map后将各结果连接起来成一个集合
xs.sum          // 数集求和
xs.product      // 数集求积
xs.max          // 最大值
xs.min          // 最小值
```

例子：列出1..M和1..N之间所有数组成的数对

``` scala
(1 to M) flatMap (x => (1 to N) map (y => (x, y)))
```

例子：求数量积

``` scala
def scalarProduct(xs: Vector[Double], ys: Vector[Double]): Double =
    (xs zip ys).map(xy => xy._1 * xy._2).sum
```

也可以用*模式匹配函数值*写成：

``` scala
def scalarProduct(xs: Vector[Double], ys: Vector[Double]): Double =
    (xs zip ys).map{ case (x, y) => x * y }.sum
```

其中，函数值

``` scala
{ case p1 => e1 … case pn => en }
```

等价于

``` scala
x => x match { case p1 => e1 … case pn => en }
```

练习：判断素数

``` scala
def isPrime(n: Int): Boolean = (2 until n) forall (d => n % d != 0)
```

练习：给出一个正整数n，找出满足1 

``` scala
val n = 10                            // > n: Int = 10
(1 until n) map (i =>
    (1 until i) map (j => (i, j)))    // > res0: scala.collection.immutable.IndexedSeq[scala.collection.immutable.IndexedSeq[(Int, Int)]] = Vector(Vector(), Vector((2,1)), Vector((3,1), (3,2)), Vector((4,1), (4,2), (4,3)), Vector((5,1), (5,2), (5,3), (5,4)), Vector((6,1), (6,2),(6,3), (6,4), (6,5)), Vector((7,1), (7,2), (7,3), (7,4), (7,5), (7,6)), Vector((8,1), (8,2), (8,3), (8,4), (8,5), (8,6), (8,7)), Vector((9,1), (9,2), (9,3), (9,4), (9,5), (9,6), (9,7), (9,8)))
```

会发现返回值类型为`IndexedSeq`，实际类型为`Vector`。这是因为`Vector`和`Range`均继承于`IndexedSeq`，而`Range`不能包含二元组，所以Scala将其换成兄弟类`Vector`。

但是期待的返回值类型为二元组的序列，上面的结果并不满足，可以这样解决（设之前结果为`xss`）：

``` scala
(xss foldRight Seq[Int]())(_ ++ _)
```

或者用内置方法`flatten`：

``` scala
xss.flatten
```

一个实用的等式：

``` scala
xs flatMap f = (xs map f).flatten
```

所以可以这样简化：

``` scala
(1 until n) flatMap (i =>
    (i until i) map (j => (i, j)))
```

最终解为：

``` scala
(1 until n) flatMap (i =>
    (i until i) map (j => (i, j))) filter ( pair =>
        isPrime(pair._1 + pair._2))
```

这解决了问题，但十分麻烦。

### `for`表达式

假设`persons`是包含类`Person`的列表，其中`Person`为

``` scala
case class Person(name: String, age: Int)
```

想取出年龄大于20岁的人，可以这样写：

``` scala
for ( p <- persons if p.age > 20 ) yield p.name
```

这和下面的相等：

``` scala
persons filter (p => p.age > 20) map (p => p.name)
```

`for`表达式和命令式语言中的循环类似，不同之处在于它将遍历结果生成了列表。

`for`表达式的形式：

``` scala
for ( s ) yield e
```

其中`s`是一系列*生成器(generator)*和*过滤器(filter)*，`e`是一个表达式，它的值会在每个遍历中返回。

* 生成器的形式为`p <- e`，其中`p`是一个模式，`e`是一个值为集合的表达式
* 过滤器的形式为`if f`，其中`f`是一个布尔表达式
* 序列必须由生成器开头
* 如果有多个生成器，后面的生成器得比前面的先改变

也可以用大括号`{}`代替小括号`()`，这时生成器和过滤器序列可以写成多行且不用带分号。

例子：用`for`表达式改写之前的练习

``` scala
for {
    i <- 1 until n
    j <- 1 until i
    if isPrime(i + j)
} yield (i, j)
```

练习：用`for`表达式重写`scalarProduct`

``` scala
def scalarProduct(xs: List[Double], ys: List[Double]): Double =
    (for ((x, y) <- xs zip ys) yield x * y).sum
```


