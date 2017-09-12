---
title: 函数式编程笔记 12
date: 2013-07-18
url: functional-programming-note-12
---

### `Stream`

想要找出从1000到10000间第二个素数，可以这样写：

``` scala
((1000 to 10000) filter isPrime)(1)
```

这性能上很不好，只需要第二个素数，但找出了1000到10000的所有素数。

可以用流来改进它，流和列表类似，但是流只在*需要时*才会被求值。

<!-- more -->

流是由一个常量`Stream.empty`和一个构造器`Stream.cons`定义：

``` scala
val xs = Stream.cons(1, Stream.cons(2, Stream.empty))
// 用Steam作工厂也可以
Stream(1, 2, 3)
// toStream将集合变为流
(1 to 1000).toStream    // > res0: Stream[Int] = Stream(1, ?)
```

来看一个和`(lo until hi).toStream`类似的函数：

``` scala
def streamRange(lo: Int, hi: Int): Stream[Int] =
    if(lo >= hi) Stream.empty
    else Stream.cons(lo, streamRange(lo + 1, hi))
```

和它的`List`版本比较：

``` scala
def listRange(lo: Int, hi: Int): List[Int] =
    if (lo > hi) Nil
    else lo :: listRange(lo + 1, hi)
```

它们的区别主要在求值方式：

* `listRange(start, end)`会生成一个包含从`end`到`start`元素的列表
* `streamRange(start, end)`会生成一个以`start`作为头元素的流
* 剩下的元素仅在需要时才会被求值，需要是指在流上调用`tail`时

流支持大部分列表上的操作，之前找素数的代码可以这样写：

``` scala
((1000 to 10000).toStream filter isPrime)(1)    // 只求值了前两个素数
```

流和列表操作上最大的区别是连接符，`x :: xs`会生成列表，而`x #:: xs`会生成流。`#::`也可以用在模式中。

这是流的trait：

``` scala
trait Stream[+A] extends Seq[A] {
    def isEmpty: Boolean
    def head: A
    def tail: Stream[A]
}
```

这是流的简单实现：

``` scala
object Stream {
    def cons[T](hd: T, tl: => Stream[T]) = new Stream[T] {
        def isEmpty = false
        def head = hd
        def tail = tl
    }
    val empty = new Stream[Nothing] {
        def isEmpty = true
        def head = throw new NoSuchElementException("empty.head")
        def tail = throw new NoSuchElementException("empty.tail")
    }
}
```

可以看到用了call-by-name参数`=>`，因此不会立刻求值。

这是`filter`方法的实现：

``` scala
class Stream[+T] {…
    def filter(p: T => Boolean): Stream[T] =
    if (isEmpty) this
    else if (p(head)) cons(head, tail.filter℗)
    else tail.filter℗
}
```

### 懒求值(Lazy Evaluation)

之前的实现有严重的性能问题，如果流的`tail`方法被调用多次，对应的流也会被多次求值。这个问题可以通过保存已计算出的值来解决，这种解决方法是可靠的，因为在纯函数式编程中无论何时求值表达式的值都是不变的。

这种方法叫做*懒求值*，相对的是*by-name求值*（每次都会被重计算）和*直接(strict)求值*（普通参数和`val`定义）。

Haskell默认使用懒求值，Scala默认使用直接求值，但也允许通过`lazy val`来应用懒求值：

``` scala
lazy val x = expr
```

练习：下面的代码的结果是什么？

``` scala
def expr = {
    val x = { print("x"); 1 }
    lazy val y = { print("y"); 2 }
    def z = { print("z"); 3 }
    z + y + x + z + y + x
}
expr
```

答案是xzyz。

之前的`Stream`实现可以改成懒求值：

``` scala
def cons[T](hd: T, tl: => Stream[T]) = new Stream[T] {
    def head = hd
    lazy val tail = tl
    …
}
```

### 无限流

例如，从给定整数开始的所有整数：

``` scala
def from(n: Int): Stream[Int] = n #:: from(n+1)
val nats = from(0)
```

例子：用筛法(The Sieve of Eratosthenes)求素数。

``` scala
object primes {
  def from(n: Int): Stream[Int] = n #:: from(n+1)        // > from: (n: Int)Stream[Int]
  def sieve(s: Stream[Int]): Stream[Int] =
    s.head #:: sieve(s.tail filter (_ % s.head != 0))    // > sieve: (s: Stream[Int])Stream[Int]
  val primes = sieve(from(2))                            // > primes: Stream[Int] = Stream(2, ?)
  (primes take 10).toList                                // > res0: List[Int] = List(2, 3, 5, 7, 11, 13, 17, 19, 23, 29)
}
```

回顾之前[用牛顿法求平方根](http://stormluke.me//post/functional-programming-note-02/)中的`sqrt`函数，可以用`Stream`来重构它：

``` scala
def sqrtStream(x: Double): Stream[Double] = {
    def improve(guess: Double) = (guess + x / guess) / 2
    lazy val guesses: Stream[Double] = 1 #:: (guess map improve)
    guess
}
…
sqrtStream(4) filter (isGoodEnough(_, 4))
```

### 总练习：倒水问题

有一个水槽和若干无刻度只知道容量的杯子，如何操作才能倒出需要的水量？

简单规划一下：

* 杯子Glass：`Int`
* 状态State：`Vector[Int]`
* 操作Moves：
    * 清空`Empty(glass)`
    * 装满`Fill(glass)`
    * 从一个杯子倒到另一个杯子`Pour(from, to)`

代码：

``` scala
class Pouring(capacity: Vector[Int]) {
  // States
  type State = Vector[Int]
  val initialState = capacity map (x => 0)
  // Moves
  trait Move {
    def change(state: State): State
  }
  case class Empty(glass: Int) extends Move {
    def change(state: State) = state updated(glass, 0)
  }
  case class Fill(glass: Int) extends Move {
    def change(state: State) = state updated(glass, capacity(glass))
  }
  case class Pour(from: Int, to: Int) extends Move {
    def change(state: State) = {
      val amount = state(from) min (capacity(to) - state(to))
      state updated(from, state(from) - amount) updated(to, state(to) + amount)
    }
  }
  val glasses = 0 until capacity.length
  val moves =
    (for (g <- glasses) yield Empty(g)) ++
      (for (g <- glasses) yield Fill(g)) ++
      (for (from <- glasses; to <- glasses if from != to) yield Pour(from, to))
  // Paths
  class Path(history: List[Move], val endState: State) {
    def extend(move: Move) = new Path(move :: history, move change endState)
    override def toString = (history.reverse mkString " ") + "--> " + endState
  }
  val initialPath = new Path(Nil, initialState)
  def from(paths: Set[Path], explored: Set[State]): Stream[Set[Path]] =
    if(paths.isEmpty) Stream.empty
    else {
      val more = for {
        path <- paths
        next <- moves map path.extend
        if !(explored contains next.endState)
      } yield next
      paths #:: from(more, explored ++ (more map (_.endState)))
    }
  val pathSets = from(Set(initialPath), Set(initialState))
  def solutions(target: Int): Stream[Path] =
    for {
      pathSet <- pathSets
      path <- pathSet
      if path.endState contains target
    } yield path
}
object test {
  val problem = new Pouring(Vector(4, 9, 19))    // > problem: Pouring = Pouring@1e86486b
  problem.moves                                  // > res0: scala.collection.immutable.IndexedSeq[Product with Serializable with problem.Move] = Vector(Empty(0), Empty(1), Empty(2), Fill(0), Fill(1), Fill(2), Pour(0,1), Pour(0,2), Pour(1,0), Pour(1,2), Pour(2,0), Pour(2,1))
  problem.solutions(17)                          // > res1: Stream[problem.Path] = Stream(Fill(0) Pour(0,2) Fill(0) Fill(1) Pour(0,2) Pour(1,2)--> Vector(0, 0, 17), ?)
}
```

当代码变长时注意：

* 尽可能地命名
* 把操作放在自然的作用域里
* 保持可优化性

### 更多资源

请直接参考网站[Functional Programming Principles in Scala](https://class.coursera.org/progfun-002/class/index)。

-完-

