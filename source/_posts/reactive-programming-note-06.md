---
title: 响应式编程笔记 06
date: 2014-06-20
url: reactive-programming-note-06
---

### 单子和副作用（Effects）

编程中四种基本的副作用为：

| 分类 | 一个 | 多个 |
| --- | --- | --- |
| 同步 | `T / Try[T]` | `Iterable[T]` |
| 异步 | `Future[T]` | `Observable[T]` |

<!-- more -->

#### 异常作为副作用（Exception as an Effect）

以一个游戏为例，在这个游戏中玩家收集金币然后买东西：

``` scala
trait Adventure {
  def collectCoins(): List[Coin]
  def buyTreasure(coins: List[Coin]): Treasure
}
```

收集金币和买东西的过程中都可能产生异常，比如被怪物吃掉或者钱不够：

``` scala
def collectCoins(): List[Coin] = {
  if (eatenByMonster(this))
    throw new GameOverException("Ooops")
  List(Gold, Gold, Silver)
}

def buyTreasure(coins: List[Coin]): Treasure = {
  if (coins.sumBy(_.value) 
    throw new GameOverException("Nice try!")
  Diamond
}
```

游戏执行的过程中可能因为一个异常没有被处理而不能继续：

``` scala
val adventure = Adventure()
val coins = adventure.collectCoins()
// 只有无异常才会继续执行
val treasure = adventure.buyTreasure(coins)
// 只有无异常才会继续执行
```

通过引入 `Try` 这个类可以让异常由隐式的变为显式的（materialize），从而更加好处理：

``` scala
abstract class Try[T]
case class Success[T](elem: T) extends Try[T]
case class Failure(t: Throwable) extends Try[Nothing]
trait Adventure {
  def collectCoins(): Try[List[Coin]]
  def buyTreasure(coins: List[Coin]): Try[Treasure]
}
val adventure = Adventure()
val coins: Try[List[Coin]] = adventure.collectCoins()
val treasure: Try[Treasure] = coins match {
  case Success(cs) => adventure.buyTreasure(cs)
  case failure @ Failure(t) => failure
}
```

其实 `Try` 是单子，被设计用来处理异常，其上定义好了许多函数式编程的基础方法：

``` scala
def flatMap[S](f: T=>Try[S]): Try[S]
def flatten[U Try[T]]: Try[U]
def map[S](f: T=>S): Try[T]
def filter(p: T=>Boolean): Try[T]
def recoverWith(f: PartialFunction[Throwable, Try[T]]): Try[T]
```

因此上面的程序也可以这样写：

``` scala
val treasure: Try[Treasure] = adventure.collectCoins().flatMap(coins => {
  adventure.buyTreasure(coins)
})
```

如果用 `for` 表达式就更简洁了：

``` scala
val treasure: Try[Treasure] = for {
  coins 
  treasure 
} yield treasure
```

`Try` 是单子，它的 `map` 和 `apply` 方法大概实现如下：

``` scala
def map[S](f: T=>S): Try[S] = this match {
  case Success(value) => Try(f(value))
  case failure @ Failure(t) => failure
}

object Try {
  def apply[T](r: =>T): Try[T] = {
    try { Success(r) }
    catch { case t => Failure(t) }
}
```

练习：下面哪个函数是正确的 `Try` 的 `flatMap` 实现？

``` scala
// a
def flatMap[S](f: T=>Try[S]): Try[S] = this match { case Success(values) => f(value)
  case failure @ Failure(t) => failure
}

// b
def flatMap[S](f: T=>Try[S]): Try[S] = this match { case Success(value) => Try(f(value))
  case failure @ Failure(t) => failure
}

// c
def flatMap[S](f: T=>Try[S]): Try[S] = this match { case Success(value) =>
  try { f(value) } catch { case t => Failure(t) } case failure @ Failure(t) => failure
}
```

答案是 c。a 中调用 `f` 时没有处理异常，b 中返回值类型为 `Try[Try[T]]`。

#### 延迟作为副作用（Latency as an Effect）

以套接字为例，从内存读取和通过网络发送数据都会产生延迟：

``` scala
trait Socket {
  def readFromMemory(): Array[Byte]
  def sendToEurope(packet: Array[Byte]): Array[Byte]
}

val socket = Socket()
val packet = socket.readFromMemory()
// 阻塞 50000 ns，不产生异常时才会继续执行
val confirmation = socket.sendToEurope(packet)
// 阻塞 150000000 ns，不产生异常时才会继续执行
```

`Future` 这个单子用来处理延迟和异常：

``` scala
import scala.concurrent._
import scala.concurrent.ExecutionContext.Implicits.global
trait Future[T] {
  def onComplete(callback: Try[T] => Unit)(implicit executor: ExecutionContext): Unit
  // alternative designs
  def onComplete(success: T => Unit, failed: Throwable => Unit): Unit
  def onComplete(callback: Observer[T]): Unit
}
// alternative designs
trait Observer[T] {
  def onNext(value: T): Unit
  def onError(error: Throwable): Unit
}
```

`implicit` 表示隐式参数。`Future` 一般在另外的线程中运行，因此回调函数 `onComplete` 会提供一个运行环境参数 `executor` 来指示运行时所使用的线程环境。

`Future` 在使用形式上可能会遇到些问题，比如：

```scala
val socket = Socket()
val packet: Future[Array[Byte]] = socket.readFromMemory()
val confirmation: Future[Array[Byte]] = packet onComplete {
  case Success(p) => socket.sendToEurope(p)
  case Failure(t) => ...
}
```

这个程序是错的，`confirmation` 的类型并不是 `Future[Array[Byte]]`。但如果这样写：

``` scala
packet onComplete {
  case Success(p) => {
    val confirmation: Future[Array[Byte]] = socket.sendToEurope(p)
  }
  case Failure(t) => ...
}
```

当延迟任务变多时程序结构会变得很混乱。

暂时先不管 `Future` 的使用形式问题，来看看如何创建一个 `Future`：

``` scala
object Future {
  def apply(body: =>T)(implicit context: ExecutionContext): Future[T]
}
```

练习：假设有以下代码

``` scala
import scala.concurrent.ExecutionContext.Implicits.global
import akka.serializer._
val memory = Queue[EMailMessage](
  EMailMessage(from = "Erik", to = "Roland"),
  EMailMessage(from = "Martin", to = "Erik"),
  EMailMessage(from = "Roland", to = "Martin"))
def readFromMemory(): Future[Array[Byte]] = Future {
  val email = queue.dequeue()
  val serializer = serialization.findSerializerFor(email)
  serializer.toBinary(email)
}
val packet: Future[Array[Byte]] = socket.readFromMemory()
packet onSuccess {
  case bs => socket.sendToEurope(p)
}
packet onSuccess {
  case bs => socket.sendToEurope(p)
}
```

当其执行完时 `email` 队列中还剩几封邮件？

答案是 2 封。尽管调用了两次 `onSuccess`，但是 `Future` 对象会缓存执行完的结果，两个 `onSuccess`都使用了同一封邮件。

#### Future 上的组合子

Scala 中的 `Future` 大概是这样：

``` scala
trait Awaitable[T] extends AnyRef {
  abstract def ready(atMost: Duration): Unit
  abstract def result(atMost: Duration): T
}

trait Future[T] extends Awaitable[T] {
  def filter(p: T => Boolean): Future[T]
  def flatMap[S](f: T => Future[S]): Future[U]
  def map[S](f: T => S): Future[S]
  def recoverWith(f: PartialFunction[Throwable, Future[T]]): Future[T]
}

object Future {
  def apply[T](body: => T): Future[T]
}
```

假设有一个 `Http` 对象用来发送网络请求，现在想向欧洲和美国均发一封邮件，代码如下：

``` scala
object Http {
  def apply(url: URL, req: Request): Future[Response] =
    {... runs the http request asynchronously ...}
}

def sendTo(url: URL, packet: Array[Byte]): Future[Array[Byte]] =
  Http(url, Request(packet)).filter(response => response.isOK).map(response => response.toByteArray)

def sendToAndBackup(packet: Array[Byte]): Future[(Array[Byte], Array[Byte])] = {
  val europeConfirm = sendTo(mailServer.europe, packet)
  val usaConfirm = sendTo(mailServer.usa, packet)
  europeConfirm.zip(usaConfirm)
}
```

但是这些方法依旧可能产生异常，程序并没有处理好这些潜在的错误。

实际上 `Future` 提供了两个用来从错误中恢复的函数：

``` scala
def recover(f: PartialFunction[Throwable,T]): Future[T]
def recoverWith(f: PartialFunction[Throwable,Future[T]]): Future[T]
```

`recover` 用一个同步的过程来恢复，而 `recoverWith` 则可以使用一个异步的过程。

现在将程序改为先向欧洲发送，若失败再向美国发送：

``` scala
def sendToSafe(packet: Array[Byte]): Future[Array[Byte]] =
  sendTo(mailServer.europe, packet) recoverWith {
    case europeError => sendTo(mailServer.usa, packet) recover {
      case usaError => usaError.getMessage.toByteArray
    }
  }
```

虽然过程上正确，但当两个网络请求都失败时返回的异常是 `usaError`，并不是想要的 `europeError`。因此想增加一个 `fallbackTo` 函数：

``` scala
def sendToSafe(packet: Array[Byte]): Future[Array[Byte]] =
  sendTo(mailServer.europe, packet) fallbackTo {
    sendTo(mailServer.usa, packet)
  } recover {
    case europeError => europeError.getMessage.toByteArray
  }

def fallbackTo(that: => Future[T]): Future[T] = {
  this recoverWith {
    case _ => that recoverWith { case _ => this }
  }
}
```

这时程序就是正确并鲁棒的了，而且 `fallbackTo` 的加入让程序变得更加直观。

练习：现在想定义一个支持 `Future` 的 `Try`：

``` scala
object Try {
  def apply(f: Future[T]): Future[Try[T]] = { ... }
}
```

它的 `apply` 方法该如何实现？

``` scala
// a
f onComplete { x => x }

// b
f recoverWith { case t => Future.failed(t) }

// c
f.map(x => Try(x))

// d
f.map(s => Success(s)) recover { case t => Failure(t) }
```

答案为 d。a 中返回值为 `Unit`，b 和 c 都只考虑了 `Try` 的一个 case。

之前看到 `Future` 继承了 `Awaitable`：

``` scala
trait Awaitable[T] extends AnyRef {
  abstract def ready(atMost: Duration): Unit
  abstract def result(atMost: Duration): T
}
```

应当注意这两个函数都是同步阻塞的，用来延迟执行或等待结果：

``` scala
val socket = Socket()
val packet: Future[Array[Byte]] = socket.readFromMemory()
val confirmation: Future[Array[Byte]] = packet.flatMap(socket.sendToSafe(_))
val c = Await.result(confirmation, 2 seconds)
println(c.toText)
```

#### Future 的应用

因为 `Future` 是单子，所以可以使用 `for` 表达式：

``` scala
val socket = Socket()
val confirmation: Future[Array[Byte]] = for {
  packet       
  confirmation 
} yield confirmation
```

现在想定义一个不断尝试执行 `Future` 最多 n 次直至成功的函数，一个可行的实现是：

``` scala
def retry(noTimes: Int)(block: => Future[T]): Future[T] = {
  if (noTimes == 0) {
    Future.failed(new Exception("Sorry"))
  } else {
    block fallbackTo {
      retry(noTimes–1) { block }
    }
  }
}
```

但是一个人曾说过：

> Recursion is the GOTO of Functional Programming - ErikMeijer

这个人其实就是本课的老师……另外一个方法就是使用 `foldRight` 或 `foldLeft` 来实现：

``` scala
// foldLeft
def retry(noTimes: Int)(block: =>Future[T]): Future[T] = {
  val ns: Iterator[Int] = (1 to noTimes).iterator
  val attempts: Iterator[Future[T]] = ns.map(_=> ()=>block)
  val failed = Future.failed(new Exception)
  attempts.foldLeft(failed)((a,block) => a recoverWith { block() })
}
// foldRight
def retry(noTimes: Int)(block: =>Future[T]): Future[T] = {
  val ns: Iterator[Int] = (1 to noTimes).iterator
  val attempts: Iterator[Future[T]] = ns.map(_=> ()=>block) val failed = Future.failed(new Exception)
  attempts.foldRight(() => failed)
  ((block, a) => () => { block() fallbackTo { a() } })
}
```

现在觉得显式的副作用并不方便，能不能将其变成隐式的呢？比如 `Future`，能否将 `T => Future[S]`变为 `T => S`？答案是可以的，Scala 提供了 `async { ... await{ ... } ... }` 这种语法结构：

``` scala
import scala.async.Async._
def async[T](body: =>T)(implicit context: ExecutionContext): Future[T]
def await[T](future: Future[T]): T
```

`await` 会将异步非阻塞的代码变成同步阻塞的，外部的 `async` 依旧返回异步非阻塞的 `Future`。

`await` 需要编译器的支持，因此需要注意以下几个方面：

* `await` 外部的 `async` 必须是直接包裹的
* `await` 不能用在 by-name 参数中
* `await` 不能用在短路布尔表达式中
* `async` 不能包含 `return` 表达式
* `await` 不能包含在 `try-catch` 中

使用了 `await` 后，代码会简洁很多：

``` scala
def retry(noTimes: Int)(block: =>Future[T]): Future[T] = async {
  var i = 0
  var result: Try[T] = Failure(new Exception("sorry man!"))
  while (i 
    result = await { Try(block) }
    i += 1
  }
  result.get
}
```

使用 `await` 重写 `filter`：

``` scala
def filter(p: T => Boolean): Future[T] = async {
  val x = await { this }
  if (!p(x)) {
    throw new NoSuchElementException()
  } else {
    x
  }
}
```

练习：如何用 `await` 实现 `flatMap`？答案如下：

``` scala
def flatMap[S](f: T => Future[S]): Future[S] =
  async { await { f( await { this } ) } }
```

如果不用 `await`，可以使用 `Promise` 实现 `filter`：

``` scala
def filter(pred: T => Boolean): Future[T] = {
  val p = Promise[T]()
  this onComplete {
    case Failure(e) =>
      p.failure(e)
    case Success(x) =>
      if (!pred(x)) p.failure(new NoSuchElementException)
      else p.success(x)
  }
  p.future
}
```

#### Promise

`Promise` 包含一个 `Future`，当调用 `Promise` 的 `complete` 方法时 `Promise` 会调用自身 `Future` 上的回调函数。`tryComplete` 和 `complete` 方法类似，只是当该 `Promise` 已经完成后 `tryComplete` 会返回 `false`。`success` 和 `failure` 是两个简化方法，用来表示成功执行或发生异常。

``` scala
trait Promise[T] {
  def future: Future[T]
  def complete(result: Try[T]): Unit
  def tryComplete(result: Try[T]): Boolean
  def success(value: T): Unit = this.complete(Success(value))
  def failure(t: Throwable): Unit = this.complete(Failure(t))
}
```

之前的 `zip` 方法可以这样实现：

``` scala
def zip[S, R](that: Future[S], f: (T, S) => R): Future[R] = {
  val p = Promise[R]()
  this onComplete {
    case Failure(e) => p.failure(e)
    case Success(x) => that onComplete {
      case Failure(e) => p.failure(e)
      case Success(y) => p.success(f(x, y))
    }
  }
  p.future
}
```

如果用 `await` 来实现的话会特别简洁：

``` scala
def zip[S, R](p: Future[S], f: (T, S) => R): Future[R] = async {
  f(await { this }, await { that })
}
```

现在用 `await` 定义一个队列函数，它会依次执行队列中的 `Future`：

``` scala
def sequence[T](fs: List[Future[T]]): Future[List[T]] = async {
  var _fs = fs
  val r = ListBuffer[T]()
  while (_fs != Nil) {
    r += await { _fs.head }
    _fs = _fs.tail
  }
  f.result
}
```

如果用 `Promise` 来定义的话是这样：

``` scala
def sequence[T](fs: List[Future[T]]): Future[List[T]] = {
  val successful = Promise[List[T]]()
  successful.success(Nil)
  fs.foldRight(successful.future) {
    (f, acc) => for { x yield x :: xs
  }
}
```


