---
title: ReactiveCocoa README 笔记
date: 2016-05-03
url: reactive-cocoa-readme
---

[ReactiveCocoa/README.md](https://github.com/ReactiveCocoa/ReactiveCocoa/blob/master/README.md)

### 序言

[FRP](https://en.wikipedia.org/wiki/Functional_reactive_programming)

> 事件流

* `Signal`
* `SignalProducer`

<!-- more -->

用来统一这些模式：

* Delegate methods
* Callback blocks
* `NSNotification`s
* Control actions and responder chain events
* Futures and promises
* Key-value observing (KVO)

### 例子: 在线搜索

#### 观察文本的编辑

``` swift
let searchStrings = textField.rac_textSignal()
    .toSignalProducer()
    .map { text in text as! String }
```

这样获取到一个发送 `String` 类型的 `SignalProducer`。为了支持 Objective-C 的 extension 方法，[目前 `as!` 是必须的](https://github.com/ReactiveCocoa/ReactiveCocoa/issues/2182)。

#### 发送网络请求

``` swift
let searchResults = searchStrings
    .flatMap(.Latest) { (query: String) -> SignalProducerNSData, NSURLResponse), NSError> in
        let URLRequest = self.searchRequestWithEscapedQuery(query)
        return NSURLSession.sharedSession().rac_dataWithRequest(URLRequest)
    }
    .map { (data, URLResponse) -> String in
        let string = String(data: data, encoding: NSUTF8StringEncoding)!
        return self.parseJSONResultsFromString(string)
    }
    .observeOn(UIScheduler())
```

* `observeOn(UIScheduler())` 在主线程上推送结果
* `flatMap(.Latest)` 保证只有最后一个网络请求是活动的

#### 获取结果

上面的代码并没有开始“运行”。直到下面这句话：

``` swift
searchResults.startWithNext { results in
    print("Search results: \(results)")
}
```

观察 `Next` 事件并处理。

#### 处理错误

在这个例子里，任何网络错误会生成 `Failed` 事件，并且会终止事件流。但上面的代码并没有处理。修复一下：

``` swift
.flatMap(.Latest) { (query: String) -> SignalProducerNSData, NSURLResponse), NSError> in
    let URLRequest = self.searchRequestWithEscapedQuery(query)
    return NSURLSession.sharedSession()
        .rac_dataWithRequest(URLRequest)
        .flatMapError { error in
            print("Network error occurred: \(error)")
            return SignalProducer.empty
        }
}
```

记录错误日志并且忽略错误。把错误换成了 `empty` 事件流。

如果出错则重试似乎更好，巧的是，确实有个 `retry` 算子。

#### 限流请求

只想在用户停止输入后执行搜索以减少流量。ReactiveCocoa 有个 `throttle` 算子。

``` swift
let searchStrings = textField.rac_textSignal()
    .toSignalProducer()
    .map { text in text as! String }
    .throttle(0.5, onScheduler: QueueScheduler.mainQueueScheduler)
```

#### 调试事件流

事件的栈轨迹会是一大坨，让调试很恶心。一种 navie 的调试方法是给流注入副作用：

``` swift
let searchString = textField.rac_textSignal()
    .toSignalProducer()
    .map { text in text as! String }
    .throttle(0.5, onScheduler: QueueScheduler.mainQueueScheduler)
    .on(event: { print ($0) }) // the side effect
```

`SignalProducer` 和 `Signal` 提供 `logEvents` 算子来自动做这件事：

``` swift
let searchString = textField.rac_textSignal()
    .toSignalProducer()
    .map { text in text as! String }
    .throttle(0.5, onScheduler: QueueScheduler.mainQueueScheduler)
    .logEvents()
```

### Objective-C 和 Swift

3.0 版本后所有的重要特性开发都集中在 Swfit API。

Objective-C API 和 Swift API 完全是分离的，但有[桥接方法](https://github.com/ReactiveCocoa/ReactiveCocoa/blob/master/Documentation/ObjectiveCBridging.md)。

### ReactiveCocoa 和 Rx 的关系？

ReactiveCocoa *故意*不是 Rx 的直接移植版。

RAC 相比于 Rx 主要用于：

* 建立更简单的 API
* Address common sources of confusion
* 更符合 Cocoa 习惯

#### 命名

在 Rx 的绝大多数版本里，流被叫做 `Observable`，是 .NET `Enumerable` 类型的并行版。另外，Rx.NET 的许多操作的名字来源于 LINQ，主要用于影射关系型数据库，比如 `Select`、`Where`。

RAC 则主要遵循 Swift 命名，比如 `map`、`filter`。其他命名的差异一般是来自 Haskell 或 Elm 中明显更好的名字。

#### Singles 和 Signal Producers (“hot” 和 “cold” observables)

Rx 里难理解的部分中的一个就是 [“hot”, “cold”, and “warm” observables](http://www.introtorx.com/content/v1.0.10621.0/14_HotAndColdObservables.html)。

简单来说，只给出如下的函数声明

``` swift
IObservable<string> Search(string query)
```

**不可能**看出订阅这个 `IObservable` 会不会引入副作用，如果它*确实*有副作用，也不能看出是否*每个订阅*会有副作用还是仅仅第一个订阅会有副作用。

这个例子展示了让 Rx（和 3.0 版前的 ReactiveCocoa）代码难一眼看懂的**真实普遍的问题**。

ReacitveCocoa 3.0 用 `Signal` 和 `SignalProducer` 来区分副作用。

换句话说，ReactiveCocoa 的改变[简约不简单](http://www.infoq.com/presentations/Simple-Made-Easy)。

#### 错误类型

RAC 允许使用 `NoError` 来*静态保证*一个事件流不允许发出异常。**这消除了许多由意外异常事件引发的 bug**。

在 Rx 里没有错误类型。

#### UI 编程

Rx 一般不知道它如何被使用，尽管用 Rx 来 UI 编程很常见。

RAC 受到很多来自 ReactiveUI 的启发，比如 `Action`。

和 ReactiveUI 不能直接改造 Rx 不同，RAC 为 UI 编程改进了很多次，即使这样会和 Rx 变得越来越不同。


