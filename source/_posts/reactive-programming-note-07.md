---
title: 响应式编程笔记 07
date: 2015-05-07
url: reactive-programming-note-07
---

### `Future[T]` 和 `Try[T]` 是[对偶（dual）](http://en.wikipedia.org/wiki/Dual_%28category_theory%29)

``` scala
trait Future[T] {
  def OnComplete[U](func: Try[T] => U)(implicit ex: ExecutionContext): Unit
}
```

<!-- more -->

对 `OnComplete` 方法的类型进行化简（`U` 化简为 `Unit`），得到

``` scala
(Try[T] => Unit) => Unit
```

翻转这个类型，得到

``` scala
Unit => (Unit => Try[T])
```

继续简化，得到

```
() => (() => Try[T]) ≈ Try[T]
```

可以看出，对方法

``` scala
def asynchronous(): Future[T] = { ... }
```

传递回调（`Try[T] => Unit`）得到 `Try[T]`，而方法

``` scala
def synchronous(): Try[T] = { ... }
```

一直阻塞直到返回 `Try[T]`。

### 同步数据流：`Iterable[T]`

这是 Scala 所有集合类型的基 trait，它定义了一个迭代器方法来一个一个地遍历集合中的元素。

``` scala
trait Iterable[T] { def iterator(): Iterator[T] }
```

迭代器是用来遍历序列元素的数据结构。它有个 `hasNext` 方法来检测下一个元素是否存在，还有个 `next` 方法来返回下一个元素。

``` scala
trait Iterator[T] { def hasNext: Boolean; def next(): T }
```

画成图是这样：

![Iterable](http://ww4.sinaimg.cn/large/6ad06ebbgw1epx0gia2dkj20vu0ecdha.jpg)

操作 `Iterable[T]` 的高阶函数有这些：

``` scala
def flatMap[B](f: A=>Iterable[B]): Iterable[B] def map[B](f: A=>B): Iterable[B]
def filter(p: A=>Boolean): Iterable[A]
def take(n: Int): Iterable[A]
def takeWhile(p: A=>Boolean): Iterable[A]
def toList(): List[A]
def zip[B](that: Iterable [B]): Iterable[(A, B)]
```

这是一个单子。

常常用弹子图（Marble Diagram）来描述这种类型。

![Marble Diagram](http://ww3.sinaimg.cn/large/6ad06ebbgw1epx0p5e16yj20t10ehach.jpg)

如果将不同命令的执行时间放大到人类级别，将会是这样：

[![Timings on human scale](http://ww3.sinaimg.cn/large/6ad06ebbgw1epx0y76ydrj20s90flq6y.jpg)](http://norvig.com/21-days.html#answers)

这时用 `Iterator` 从磁盘中读取文件会让程序阻塞很长时间：

``` scala
def ReadLinesFromDisk(path: String): Iterator[String] = {
  Source.fromFile(path).getLines()
}

val lines = ReadLinesFromDisk("\c:\tmp.txt")
for (line 
  ... DoWork(line) ...
}
// 2 weeks per line.
```

现在用之前的对偶化技巧将拉（pull）模型转化为推（push）模型。

第零步，化简。将之前的签名：

``` scala
trait Iterable[T] {
  def iterator(): Iterator[T]
}

trait Iterator[T] {
  def hasNext: Boolean
  def next(): T
}
```

抽象出类型：

``` scala
() => (() => Try[Option[T]])
```

* `() => ( ... )` 由 `iterator()` 而来，
* `() => Try[Option[T]]` 由 `next()` 而来，
* `Option` 表示了 `hasNext`，
* `Try` 显式化了错误。

第一步，翻转。

``` scala
() => (() => Try[Option[T]])
```

翻转为：

``` scala
(Try[Option[T]] => Unit) => Unit
```

第二步，化简。将组合在一起的类型拆分为三个：

``` scala
( T => Unit,
  Throwable => Unit,
  () => Unit
) => Unit
```

第三步，复杂化。得出对应的签名：

``` scala
trait Observable[T] {
  def Subscribe(observer: Observer[T]): Subscription
}

trait Observer[T] {
  def onNext(value: T): Unit
  def onError(error: Throwable): Unit
  def onCompleted(): Unit
}

trait Subscription {
   def unsubscribe(): Unit
}
```

通过对比可发现 `Iterable[T]` 和 `Observable[T]` 是对偶。

### 对比 `Future` 和 `Observable`

首先看签名：

``` scala
Observable[T] = (Try[Option[T]] => Unit) => Unit
Future[T]     = (Try[      [T]] => Unit) => Unit
```

`Observable[T]` 多了 `Option`，这使其可以处理多次数据。

并发方面有什么不同呢？

``` scala
object Future {
  def apply[T](body: => T)(implicit executor: ExecutionContext): Future[T]
}

trait Observable[T] {
  def observeOn(scheduler: Scheduler): Observable[T]
}
```

`Future` 只执行一次，仅需要当前线程相关的 `ExecutionContext`，而 `Observable` 执行多次，需要一个 `Scheduler` 来控制。

### `Observable` 基础

来看一个使用 `Observable` 的例子：

``` scala
val ticks: Observable[Long] = Observable.interval(1 seconds)
val evens: Observable[Long] = ticks.filter(s=>s%2==0)
val bufs: Observable[Seq[Long]] = ticks.buffer(2,1)
val s = bufs.subscribe(b=>printLn(b))
readLine()
s.unscubscribe()
```

分步执行如下：

``` scala
val ticks: Observable[Long] = Observable.interval(1 seconds)
```

![Observable-eg-01](http://ww3.sinaimg.cn/large/6ad06ebbgw1epx1vfkdbzj20qo02kwer.jpg)

``` scala
val evens: Observable[Long] = ticks.filter(s=>s%2==0)
```

![Observable-eg-02](http://ww2.sinaimg.cn/large/6ad06ebbgw1epx1vrs8z6j20qi02kmxi.jpg)

``` scala
val bufs: Observable[Seq[Long]] = ticks.buffer(2,1)
```

![Observable-eg-03](http://ww3.sinaimg.cn/large/6ad06ebbgw1epx1xgfogjj20qq02x3yx.jpg)

练习：

``` scala
val xs = Observable.range(1, 10)
```

的弹子图如下：

![range](http://ww1.sinaimg.cn/large/6ad06ebbgw1epx205cuvoj20qq02maac.jpg)

那么

``` scala
val ys = xs.map(x => x + 1)
```

的弹子图是什么？

![options](http://ww4.sinaimg.cn/large/6ad06ebbgw1epx20fkytoj20zp0asac4.jpg)

答案是 B。

### `Observable` 上的组合子

操作 `Observable[T]` 的高阶函数有这些：

``` scala
def flatMap[B](f: A=>Observable[B]): Observable[B]
def map[B](f: A=>B): Observable[B]
def filter(p: A=>Boolean): Observable[A]
def take(n: Int): Observable[A]
def takeWhile(p: A=>Boolean): Observable[A]
def toList(): List[A]
def zip[B](that: Observable[B]): Observable[(A, B)]
```

其中 `map` 的弹子图是：

![map](http://ww4.sinaimg.cn/large/6ad06ebbgw1epx3yeqsnaj20th0ehdi5.jpg)

`flatMap` 定义如下：

``` scala
def flatMap(f: T=>Observable[S]): Observable[S] = { map(f).flatten() }
```

其弹子图为：

![flatMap](http://ww4.sinaimg.cn/large/6ad06ebbgw1epx416dmvkj20zq0e6acj.jpg)

有两种扁平化叠套流的方法，一种是 `flatten`：

``` scala
val xs: Observable[Int] = Observable(3,2,1)
val yss: Observable[Observable[Int]] =
   xs.map(x => Observable.Interval(x seconds).map(_=>x).take(2))
val zs: Observable[Int] = yss.flatten()
```

![flatten](http://ww2.sinaimg.cn/large/6ad06ebbgw1epx45oxj4rj20z80aigna.jpg)

![merge](http://ww3.sinaimg.cn/large/6ad06ebbgw1epx478ezznj20t60htmzd.jpg)

另一种是 `concat`：

``` scala
val xs: Observable[Int] = Observable(3,2,1)
val yss: Observable[Observable[Int]] =
   xs.map(x => Observable.Interval(x seconds).map(_=>x).take(2))
val zs: Observable[Int] = yss.concat()
```

![concat-eg](http://ww2.sinaimg.cn/large/6ad06ebbgw1epx492aeysj20ys0ah0uk.jpg)

![concat](http://ww3.sinaimg.cn/large/6ad06ebbgw1epx49b2cuuj20t20hpdib.jpg)

下面通过一个处理地震通知的例子来展示如何映射和过滤异步的数据流。定义基本的结构如下：

``` scala
def usgs(): Observable[EarthQuake] = { ... }

class EarthQuake {
  ...
  def magnitude: Double
  def location: GeoCoordinate
}

object Magnitude extends Enumeration {
  def apply(magnitude: Double): Magnitude = { ... }
  type Magnitude = Value
  val Micro, Minor, Light, Moderate, Strong, Major, Great = Value
}
```

用起来大概是这样：

``` scala
val quakes = usgs()
val major = quakes
  .map(q=>(q.Location, Magnitude(q.Magnitude)))
  .filter{ case (loc,mag) => mag >= Major }
major.subscribe({ case (loc, mag) => {
  println($"Magnitude ${ mag } quake at ${ loc }")
})
```

现在想通过网络将地震处的地理坐标转换为国家信息：

``` scala
def reverseGeocode(c: GeoCoordinate): Future[Country] = { ... }
val withCountry: Observable[Observable[(EarthQuake, Country)]] =
  usgs().map(quake => {
    val country: Future[Country] = reverseGeocode(q.Location)
    Observable(country.map(country=>(quake,country)))
  })
// This
val merged: Observable[(EarthQuake, Country)] = withCountry.flatten()
// Or this?
val merged: Observable[(EarthQuake, Country)] = withCountry.concat()
```

那么问题来了，该用 `flatten` 还是 `concat`？如果用 `flatten`：

![geo-flatten](http://ww4.sinaimg.cn/large/6ad06ebbgw1epx4ru2yr7j20vu0fvjtc.jpg)

最终收到地震消息的顺序会因为反向解析的延迟而出现错乱，而如果用 `concat` 则没问题：

![geo-concat](http://ww4.sinaimg.cn/large/6ad06ebbgw1epx4trgmxqj210v0eyq54.jpg)

看一个新函数 `groupBy`：

``` scala
def groupBy[K](keySelector: T=>K): Observable[(K,Observable[T])]
```

它的弹子图为：

![groupBy](http://ww4.sinaimg.cn/large/6ad06ebbgw1epx4v4pqr0j20pp0f8dhn.jpg)

原序列元素根据形状分为了两组，最终产生了三个数据流。

现在想让收到的地震信息根据国家不同而分类，可以这样写：

``` scala
val merged: Observable[(EarthQuake, Country)] = withCountry.flatten()
val byCountry: Observable[(Country, Observable[(EarthQuake, Country)]] =
  merged.groupBy{ case (q,c) => c }
```

![group-eg](http://ww3.sinaimg.cn/large/6ad06ebbgw1epx4ylfq96j20xx0a3gmt.jpg)

练习：

若想统计不同国家发生地震的平均次数，部分代码如下：

``` scala
val byCountry: Observable[(Country, Observable[(EarthQuake, Country)]]
def runningAverage(s : Observable[Double]): Observable[Double] = {...}
val runningAveragePerCountry : Observable[(Country, Observable[Double])]
```

那么 `runningAveragePerCountry` 的实现应该是什么？

``` scala
// a)
val runningAveragePerCountry = byCountry.map{
  case (country, quakes) => (country, runningAverage(quakes))
}

// b)
val runningAveragePerCountry = byCountry.map{
  case (country, quakes) => (country, runningAverage(quakes.map(_.Magnitude))
}

// c)
val runningAveragePerCountry = byCountry.map{
  case (country, cqs) => (country, runningAverage(cqs.map(_._1.Magnitude))
}
```

根据类型匹配的原则可以得出答案为 C。

### 订阅

如何取消订阅呢？这样：

``` scala
val quakes: Observable[EarthQuake] = ...
val s: Subscription = quakes.Subscribe(...)
s.unsubscribe()
```

`Observable` 可分为两种，一种称为 Hot Observable，所有的订阅者共享同样的源：

![hot observable](http://ww4.sinaimg.cn/large/6ad06ebbgw1epx6vvjgxqj20ih0hwq4k.jpg)

另一种称为 Cold Observable，每个订阅者都有自己的私有源：

![cold observable](http://ww4.sinaimg.cn/large/6ad06ebbgw1epx6w6gawnj20ji0hxgni.jpg)

注意取消订阅不等于终止源，因为可能还存在其他订阅者。

`Subscription` 的基础定义如下：

``` scala
trait Subscription {
   def unsubscribe(): Unit
}

object Subscription {
  def apply(unsubscribe: => Unit):Subscription
}
```

`Subscription` 家族中包含这些成员：

``` scala
trait BooleanSubscription extends Subscription {
  def isUnsubscribed: Boolean
}

trait CompositeSubscription extends BooleanSubscription {
  def +=(s: Subscription): this.type
  def -=(s: Subscription): this.type
}

trait MultipleAssignmentSubscription extends BooleanSubscription {
  def subscription: Subscription
  def subscription_=(that: Subscription): this.type
}
```

下面的代码中 `subscription` 被调用了两次：

``` scala
val subscription = Subscription {
   println("bye, bye, I’m out fishing")
}
subscription.unsubscribe()
subscription.unsubscribe()
```

结果是只有第一次会输出字符串。就是说，`unsubscribe` 可以被调用多次，它必须是[幂等（idempotent）](http://zh.wikipedia.org/wiki/%E5%86%AA%E7%AD%89)的。

`BooleanSubscription` 有一个 `isUnsubscribed` 方法，它能指示此订阅是否已被取消：

``` scala
val subscription = BooleanSubscription {
   println("bye, bye, I’m out fishing")
}
println(subscription.isUnsubscribed)
subscription.unsubscribe()
println(subscription.isUnsubscribed)
```

![BooleanSubscription](http://ww3.sinaimg.cn/large/6ad06ebbgw1epx7ctkstbj204q0f2glw.jpg)

`CompositeSubscription` 可以包含许多订阅，当其被取消时所包含的订阅也会被取消：

``` scala
val a = BooleanSubscription { println("A") }
val b = Subscription { println("B") }
val composite = CompositeSubscription(a,b)
println(composite.isUnsubscribed)
composite.unsubscribe()
println(composite.isUnsubscribed)
println(a.isUnsubscribed)
composite += Subscription{ println ("C") }
```

![CompositeSubscription-subscribe](http://ww2.sinaimg.cn/large/6ad06ebbgw1epx7dcsud4j20dr0ipgna.jpg)

当新加入订阅时，若 `CompositeSubscription` 未被取消则新订阅状态不变，若 `CompositeSubscription` 已被取消则新订阅会被立刻取消：

![CompositeSubscription-add](http://ww4.sinaimg.cn/large/6ad06ebbgw1epx7e96l9tj210h0h0aec.jpg)

`MultiAssignment` 只能包含一个子订阅，且它自身包含了一个隐式的订阅：

![MultiAssignment-subscribe](http://ww3.sinaimg.cn/large/6ad06ebbgw1epx7hz7kt6j20zv0h5q7a.jpg)

新加入订阅时的行为和 `CompositeSubscription` 类似：

![MultiAssignment-add](http://ww3.sinaimg.cn/large/6ad06ebbgw1epx7kqh7uaj20za0gigpe.jpg)

当子订阅被取消时，`MultiAssignment` 的隐式订阅并不会被取消：

![MultiAssignment-unscribe](http://ww3.sinaimg.cn/large/6ad06ebbgw1epx7llgmrrj20yt0f2wh8.jpg)

练习：

有如下代码段：

``` scala
val a = BooleanSubscription { println("A") }
val b = Subscription { println("B") }
val c = CompositeSubscription(a,b)
val m = MultiAssignmentSubscription()
m.subscription = c
c.unsubscribe
```

下面哪个是正确的？

``` scala
a) b.isUnsubscribed == true
b) a.isUnsubscribed == false
c) m.isUnsubscribed == true
d) c.isUnsubscribed == true
```

答案是 D。


