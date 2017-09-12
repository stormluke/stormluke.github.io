---
title: 响应式编程笔记 08
date: 2015-05-08
url: reactive-programming-note-08
---

## 创建 `Observable`

所有的 `Observable` 工厂方法都是由下面这个派生而来：

``` scala
object Observable {
  def apply[T](s: Observer[T] 􏰀 Subscription): Observable[T]
}
```

<!-- more -->

![Observable.create](http://ww1.sinaimg.cn/large/6ad06ebbgw1epy87ziblyj20vw0a9jtb.jpg)

首先是两个最简单的：`never` 和 `error`

![never](http://ww2.sinaimg.cn/large/6ad06ebbgw1epy8canoq9j20x507zt8z.jpg)

![error](http://ww2.sinaimg.cn/large/6ad06ebbgw1epy8ddjvggj20xd07q74o.jpg)

``` scala
def never(): Observable[Nothing] = Observable[Nothing](observer =>􏰀 {
  Subscription {}
})

def apply[T](error: Throwable): Observable[T] =
  Observable[T](observer 􏰀=> {
    observer.onError(error)
    Subscription {}
  })
```

然后是 `startWith`，它能在流之前添加元素：

![startWith](http://ww1.sinaimg.cn/large/6ad06ebbgw1epy8gethuwj20t70dwwgo.jpg)

``` scala
def startWith(ss: T*): Observable[T] = {
  Observable[T](observer 􏰀=> {
    for(s <- ss) observer.onNext(s)
    subscribe(observer)
  })
})
```

`filter`：

![filter](http://ww4.sinaimg.cn/large/6ad06ebbgw1epy8j13td9j20q10d275n.jpg)

``` scala
def filter(p: T 􏰀=> Boolean): Observable[T] = {
  Observable[T](observer 􏰀=> {
    subscribe (
      (t: T) 􏰀=> { if(p(t)) observer.onNext(t) },
      (e: Throwable) 􏰀=> { observer.onError(e) },
      () 􏰀=> { observer.onCompleted() }
    )
  })
}
```

`map`：

![map](http://ww4.sinaimg.cn/large/6ad06ebbgw1epy8lchhfcj20qd0d0jt9.jpg)

``` scala
def map[S](f: T 􏰀=> S): Observable[S] = {
  Observable[S](observer 􏰀=> {
    subscribe (
      (t: T) => { observer.onNext(f(t)) },
      (e: Throwable) => { observer.onError(e) },
      () => { observer.onCompleted() }
    )
  })
}
```

和 `Iterable` 的 `map` 对比下：

``` scala
def map[S](f: T => S): Iterable[S] = {
  new Iterable[S] {
    val it = this.iterator()
    def iterator: Iterator[S] = new Iterator[S] {
      def hasNext: Boolean = { it.hasNext }
      def next(): S = { f(it.next()) }
    }
  }
}
```

## `Subject`

现在想将一个 `Future[T]` 转化成 `Observable[T]`：

![future-to-observable](http://ww2.sinaimg.cn/large/6ad06ebbgw1epy8qf2k1xj20su0ep75a.jpg)

为了要达成这个目的，需要引入一个新类型：`Subject`。回顾下 `Promise` 是如何做的：

``` scala
def map[S](f: T => S)(implicit executor: ExecutionContext): Future[S] = {
  val p = Promise[S]()
  onComplete {
    case result => {... p.complete(E) ...}
  }(executor)
  p.future
}
```

当在 `Promise` 上调用 `complete` 时，`Promise` 会调用内部的 `Future` 上的 `onComplete`：

![promise](http://ww1.sinaimg.cn/large/6ad06ebbgw1epy8y1okjwj20yk0c2abn.jpg)

`Subject[T]` 和 `Promise[T]` 所担任的角色类似，它同时包含 `Observer[T]` 和 `Observable[T]`，既可以在 `Observer[T]` 上调用 `onNext` 等方法，也能 `subscribe` 一个 `Observable[T]`。可以说 `Subject` 让 Cold Observable 变成了 Hot Observable：

![subject](http://ww1.sinaimg.cn/large/6ad06ebbgw1epyagmpawdj20yi0gn41q.jpg)

来看几个不同类型的 `Subject` 的例子：

``` scala
val channel = PublishSubject[Int]()
val a = channel.subscribe(x => println("a: "+x))
val b = channel.subscribe(x => println("b: "+x)) channel.onNext(42)
a.unsubscribe()
channel.onNext(4711)
channel.onCompleted()
val c = channel.subscribe(x􏰀println("c: "+x))
channel.onNext(13)
// a: 42; b:42, 4711, !; c: !
```

`PublishSubject` 是基础行为的 `Subject`，当订阅一个已经停止的 `PublishSubject` 时，会直接收到 `onCompleted()`。

``` scala
val channel = ReplaySubject[Int]()
val a = channel.subscribe(x => println("a: "+x))
val b = channel.subscribe(x => println("b: "+x))
channel.onNext(42)
a.unsubscribe()
channel.onNext(4711)
channel.onCompleted()
val c = channel.subscribe(x􏰀println("c: "+x))
channel.onNext(13)
// a: 42; b: 42, 4711, !; c: 42, 4711, !
```

`ReplaySubject` 会缓存所有元素，当订阅一个已经停止的 `ReplaySubject` 时，会收到所有的元素。

还用两种 `Subject`，一个是 `BehaviorSubject`，它会缓存最后一个元素，当订阅一个已经停止的 `BehaviorSubject` 时，会收到最后一个元素；另一个是 `AsyncSubject`，它也会缓存最后一个元素，但不论何时订阅，都只能收到最后一个元素。下面是它们的对比图：

![subjects](http://ww4.sinaimg.cn/large/6ad06ebbgw1epyavfalrgj20yq0gmdiy.jpg)

练习：

下面这段代码：

``` scala
val channel = AsyncSubject[Int]()
val a = channel.subscribe(x => println("a: "+x))
val b = channel.subscribe(x => println("b: "+x))
channel.onNext(42)
a.unsubscribe()
channel.onNext(4711)
channel.onCompleted()
val c = channel.subscribe(x􏰀println("c: "+x))
channel.onNext(13)
```

哪个频道输出是正确的？

``` scala
a: 4711, !
b: 42, 4711, !
b: 4711, !
```

答案是 b 频道。

于是就可以这样来将 `Future[T]` 转化成 `Observable[T]`：

``` scala
object Observable {
  def apply[T](f: Future[T]): Observable[T] = {
    val subject = AsyncSubject[T]()
    f onComplete {
      case Failure(e) 􏰀 { subject.onError(e) }
      case Success(c) 􏰀 { subject.onNext(c); subject.onCompleted() }
    }
    subject
  }
}
```

## `Notification`

想显式处理流中的错误怎么办？可以用 `Notification`：

``` scala
abstract class Try[+T]
case class Success[T](elem: T) extends Try[T]
case class Failure(t: Throwable) extends Try[Nothing]
abstract class Notification[+T]
case class OnNext[T](elem: T) extends Notification[T]
case class OnError(t: Throwable) extends Notification[Nothing]
case object OnCompleted extends Notification[Nothing]
def materialize: Observable[Notification[T]] = { ... }
```

弹子图如下：

![Notification](http://ww2.sinaimg.cn/large/6ad06ebbgw1epybkdyh2qj20r20dr0ub.jpg)

## 阻塞 `Observable`

可以通过 `Observable.toBlockingObservable()` 或者 `BlockingObservable.from()` 方法将非阻塞的 `Observable` 变为阻塞的。但应注意 Rx 中所有的操作都是非阻塞的：

``` scala
val xs: Observable[Long] = Observable.interval(1 second).take(5)
val ys: List[Long] = xs.toBlockingObservable.toList
println(ys)
println("bye")
val zs: Observable[Long] = xs.sum
val s: Long = zs.toBlockingObservable.single
```

## 归约 `Observable`

可以使用 `reduce` 方法来创建标量 `Observable`：

``` scala
def reduce(f: (T, T) => T): Observable[T]
```

![reduce](http://ww1.sinaimg.cn/large/6ad06ebbgw1epyhpkrhd1j20rb0ec760.jpg)

## `Scheduler`

如何设计一个将 `Iterable` 转变成 `Observable` 的函数？这个 `from` 函数要支持无限流，并且能取消订阅：

![from](http://ww3.sinaimg.cn/large/6ad06ebbjw1epzsukxgj6j20ty0exq51.jpg)

``` scala
def from[T](seq: Iterable[T]) : Observable[T] = { ... }
val infinite: Iterable[Int] = nats()
val subscription = from(infinite). subscribe(x => println(x))
subscription.unsubscribe()
```

注意 `Iterable` 是懒求值的：

``` scala
def nats(): Iterable[Int] = new Iterable[Int] {
  var i = -1
  def iterator: Iterator[Int] = new Iterator[Int] {
    def hasNext: Boolean = { true }
    def next(): Int = { i +=1; i }
  }
}
```

由于流可能是无限的，直接用 `foreach` 不行：

``` scala
def from[T](seq: Iterable[T]) : Observable[T] = { Observable(observer => {
seq.foreach(s => observer.onNext(s)) observer.onCompleted()
  // we never get here
  Subscription{}
  })
}
val infinite: Iterable[Integer] = nats()
val subscription = from(infinite).subscribe(x => println(x))
// hence we never get here
subscription.unsubscribe()
```

必须在其他的线程中执行生成器，这时就需要 `Scheduler` 了：

``` scala
trait Scheduler {
  def schedule(work: =>Unit): Subscription
}

val scheduler = Scheduler.NewThreadScheduler
val subscription = scheduler.schedule {
  println("hello world");
}
```

在 `Scheduler` 中使用 `foreach` 虽然不会阻塞当前线程，但不能取消订阅：

``` scala
def from[T](seq: Iterable[T])(implicit scheduler: Scheduler): Observable[T] = {
  Observable[T](observer => {
    scheduler.schedule {
      seq.foreach(s ⇒ observer.onNext(s))
      observer.onCompleted()
    }
  })
}
```

需要演化出两个更精细的控制方法：

``` scala
trait Scheduler {
  def schedule(work: =>Unit): Subscription
  def schedule(work: Scheduler=>Subscription): Subscription
  def schedule(work: (=>Unit)=>Unit): Subscription
}
```

这次就没问题了：

``` scala
def from[T](seq: Iterable[T])(implicit scheduler: Scheduler): Observable[T] = {
  Observable[T](observer => {
    val it = seq.iterator()
    scheduler.schedule(self => {
      if (it.hasnext) { observer.onNext(it.next()); self() }
      else { observer.onCompleted() }
    })
  })
}
```

其中

``` scala
if (it.hasnext) { observer.onNext(it.next()); self() }
```

的调用过程大概是这样：

![self-call](http://ww1.sinaimg.cn/large/6ad06ebbjw1epzt8u4pkdj20vk0e2dim.jpg)

那么 `schedule` 到底是如何定义的呢？如下：

``` scala
def schedule(work: (=>Unit)=>Unit): Subscription = {
  val subscription = new MultipleAssignmentSubscription();
  schedule(scheduler => {
    loop(scheduler, work, subscription);
    subscription;
  })
}

def loop(s: Scheduler, w: (=>Unit)=>Unit), m: MultipleAssignmentSubscription): Unit = {
   m.Subscription = s.schedule { w { loop(s, w, m) } };
}
```

把 `loop` 整理到 `schedule` 内部，就变成了：

``` scala
def schedule(work: (⇒Unit)⇒Unit): Subscription = {
  val subscription = new MultipleAssignmentSubscription()
  schedule(scheduler => {
    def loop(): Unit = {
      subscription.Subscription = scheduler.schedule {
        work { loop() }
      }
    }
    loop()
    subscription
   })
}
```

现在想将 `Scheduler` 转化成 `Observable[Unit]`，实现如下：

``` scala
object Observable {
  def apply()(implicit scheduler: Scheduler): Observable[Unit] = {
    Observable(observer => {
      scheduler.schedule(self => {
          observer.OnNext(())
          self()
      })
    })
  }
}

implicit val scheduler = Scheduler.NewThreadScheduler
val ticks: Observable[Unit] = Observable()
```

## Rx 约定

`subscribe` 的实现可以概括成这样：

``` scala
object Observable {
  def apply(s: Observer[T]=>Subscription) = new Observable[T] {
    def subscribe(o: Observer[T]): Subscription = { Magic(s(o)) }
  }
}
```

就是说

``` scala
val s = Observable(o=>F(o)).subscribe(observer)
```

在概念上和

``` scala
val s = Magic(F(observer))
```

相同。`Magic` 所做的工作是当在 `F` 中调用 `observer.onCompleted` 或 `observer.onError` 时，它会取消 `s` 的订阅。

就是说，流只能以多个 `onNext` 后接零个或多个 `onCompleted` 或 `onError` 构成。这是 Rx 的约定，表示为 `(onNext)*(onCompleted+onError)?`。

注意，绝不要自己实现 `Observable[T]` 或 `Observer[T]`，而应该始终使用 `Observable(...)` 和 `Observer(...)` 工厂方法。


