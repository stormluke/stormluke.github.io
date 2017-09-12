---
title: WWDC 2014 笔记 - Swift
date: 2014-06-18
url: wwdc-2014-note-swift
---

磨蹭到现在才看了一部分的 Session，有些收获，整理一些关于 Swift 语言的内容，主要来自以下几个 Session：

* [402 Introduction to Swift](https://developer.apple.com/videos/wwdc/2014/?id=402)
* [403 Intermediate Swift](https://developer.apple.com/videos/wwdc/2014/?id=403)
* [404 Advanced Swift](https://developer.apple.com/videos/wwdc/2014/?id=404)
* [406 Integrating Swift with Objective C](https://developer.apple.com/videos/wwdc/2014/?id=406)
* [407 Swift Interoperability in Depth](https://developer.apple.com/videos/wwdc/2014/?id=407)
* [408 Swift Playgrounds](https://developer.apple.com/videos/wwdc/2014/?id=408)
* [409 Introduction to LLDB and The Swift REPL](https://developer.apple.com/videos/wwdc/2014/?id=409)
* [410 Advanced Swift Debugging in LLDB](https://developer.apple.com/videos/wwdc/2014/?id=410)

<!-- more -->

Apple 已经提供了两种清晰度视频和相应 PDF 的直接下载。另外 [GitHub](https://github.com/search?q=WWDC) 上也有不少工具能批量下载，其中 [WWDC-Downloader](https://github.com/jfahrenkrug/WWDC-Downloader) 可以下载 Sample Code。

下面按照 Session 顺序来做些笔记。

### Session 402 - Introduction to Swift

看名字就知道不会有啥干货，基本上是概述了 Swift 的各个主要特性，均能在 iBooks 书店里的那本 *The Swift Language* 中找到。

### Session 403 - Intermediate Swift

主要介绍了以下几个内容：

* Optionals
* 内存管理
* 构造器
* 闭包
* 模式匹配

虽然都能在 *The Swift Language* 中找到这些主题，但还是有些值得记下。

#### Optionals

Swift 是类型安全（type safe）语言，为了强化这一点，Swift 把下面这坨东西

![old-null](http://ww4.sinaimg.cn/large/6ad06ebbgw1ehi3u5k6i5j20il0873ys.jpg)

统一为 Optionals。Swift 支持 Optional Types，表示一个类型的值可能为空，比如 `Int?`、`String?`、`MyObject?` 之类。在类型名后加 `?` 表示这是一个 Optional Type，因为 Optional Type 值不确定，所以必须声明为变量 `var`。

Optional Type 有几种解包方式。在变量名后加 `!` 表示强制解包，若原值为 `nil` 则会出现运行时错误。较好的方式是使用 `if let` 语句，可以同时测试原值是否为空并解包：

``` swift
var neighbors = ["Alex", "Anna", "Madison", "Dave"] let index = findIndexOfString("Anna", neighbors)
if let indexValue = index { // index is of type Int?
  println("Hello, \(neighbors[indexValue])")
} else {
  println("Must've moved away")
}
```

其中若 `index` 有实际值，则 `indexValue` 会隐式解包并绑定这个值，并且类型为实际类型，在本例中类型为 `Int`。

`if let` 虽然可以很好地处理一个 Optional Type，但面对多级关系时就会产生嵌套：

``` swift
var addressNumber: Int?
if let home = paul.residence {
  if let postalAddress = home.address {
    if let building = postalAddress.buildingNumber {
      if let convertedNumber = building.toInt() {
        addressNumber = convertedNumber
      }
    }
  }
}
```

因此有了 Optional Chaining：

``` swift
addressNumber = paul.residence?.address?.buildingNumber?.toInt()
```

若中间出现 `nil`，则会立即停止并返回 `nil`。

![optional-chaining](http://ww2.sinaimg.cn/large/6ad06ebbgw1ehi4qmgfejj20za07q0u8.jpg)

其实 Optional 是一个枚举类型：

``` swift
enum Optional<T> {
  case None
  case Some(T)
}
```

Session 中没提到的是若在类型名后加 `!`，则会将传入的 Optional Type 隐式解包，比如：

``` swift
var optionalStr: String? = "Hello"  // {Some "Hello"}
var unwrappedStr: String! = str     // "Hello"
```

#### 内存管理

Swift 也使用 ARC 做内存管理，所有的引用默认均为强引用，然后就会遇到和 Objective-C 一样的问题。

首先应分清引用的所有权（Ownership），用房间和租户的例子来说明：

``` swift
class Apartment {
  var tenant: Person?
}

class Person {
  var home: Apartment?
  func moveIn(apt: Apartment) {
    self.home = apt
    apt.tenant = self
  }
}

var renters = ["Elsvette": Person()]
var apts = [507: Apartment()]
renters["Elsvette"]!.moveIn(apts[507]!)
```

这时的引用关系为：

![ownership-strong](http://ww2.sinaimg.cn/large/6ad06ebbgw1ehi68hf4oij20dn09q74p.jpg)

可以看到产生了引用循环。解决方法是在变量声明前加上 `weak` 修饰符，指明这是一个弱引用：

``` swift
class Apartment {
  weak var tenant: Person?
}

class Person {
  weak var home: Apartment?
  func moveIn(apt: Apartment) {
    self.home = apt
    apt.tenant = self
  }
}

var renters = ["Elsvette": Person()]
var apts = [507: Apartment()]
renters["Elsvette"]!.moveIn(apts[507]!)
```

此时引用关系为：

![ownership-weak-1](http://ww3.sinaimg.cn/large/6ad06ebbgw1ehi6bft5o6j20e509l74p.jpg)

当执行 `renters["Elsvette"]!.moveIn(apts[507]!)` 后，`renters` 取消对 `Person` 实例的强引用，之后 ARC 释放所有无强引用指向的实例，完成内存回收：

![ownership-weak-2](http://ww3.sinaimg.cn/large/6ad06ebbgw1ehi6ll3jc2j207i0963yk.jpg)

关于弱引用有几个需要注意的地方：

* 弱引用一定是 Optional 值
* 对 Optional 进行 `if let` 绑定时会产生强引用
* `if` 测试弱引用时不会产生强引用
* Optional Chaining 在方法调用间不会保存强引用

由于弱引用一定是 Optional，而 Optional 必须为 `var` 变量，在表示同生命周期的关系时可能会遇到无法用 `weak` 修饰 `let` 常量声明的问题，比如信用卡和持有者间的关系：

``` swift
class Person {
  var card: CreditCard?
}

class CreditCard {
  let holder: Person
  init(holder: Person) {
    self.holder = holder
  }
}
```

解决方式是用 `unowned` 代替 `weak` 来修饰 `let`：

``` swift
class Person {
  var card: CreditCard?
}

class CreditCard {
  unowned let holder: Person
  init(holder: Person) {
    self.holder = holder
  }
}

var renters = ["Elsvette": Person()]
customers["Elsvette"]!.saveCard()
```

此时的引用图为：

![unowned](http://ww3.sinaimg.cn/large/6ad06ebbgw1ehi7tbngvjj208a0ev74n.jpg)

当执行完 `customers["Elsvette"] = nil`，`renters` 取消对 `Person` 实例的引用，之后 ARC 释放 `Person` 实例和 `CreditCard` 实例，完成内存回收。

那么该如何合理使用这三种引用关系（`strong`、`weak`、`unowned`）呢？

* 在所有者指向它所拥有的对象时使用 `strong`

![strong](http://ww1.sinaimg.cn/large/6ad06ebbgw1ehi8arwgg4j20e10alwem.jpg)

* 在生命周期互相独立的对象间使用 `weak`

![weak](http://ww4.sinaimg.cn/large/6ad06ebbgw1ehi8coq84aj20cs0adq32.jpg)

* 在生命周期相同的关系中，被拥有者指向它的拥有者时使用 `unowned`

![unowned](http://ww3.sinaimg.cn/large/6ad06ebbgw1ehi8cyssxrj204b0a6q2u.jpg)

#### 构造器

Swift 要求所有的值必须被初始化之后才能使用，因此要注意在构造器中要先初始化完所有成员才能执行它们之上的方法，下面的程序是错的：

![init-error](http://ww1.sinaimg.cn/large/6ad06ebbgw1ehjfia00tfj20kt0bcta6.jpg)

另外一个值得注意的地方是父类构造器方法调用的位置，如下例所示：

![init-override-error](http://ww2.sinaimg.cn/large/6ad06ebbgw1ehjg4f1ip8j20uk0hq76n.jpg)

由于重载了 `fillGasTank` 方法，而子类的这个方法中使用了未初始化的 `hasTurbo` 成员变量导致错误。正确的做法是把 `super.init` 放在子类构造器方法的最后调用：

![init-override](http://ww3.sinaimg.cn/large/6ad06ebbgw1ehjjgoal22j20hp08lt9n.jpg)

Swift 中提供了 `convenience` 来修饰为便利性而引入的构造方法，各个构造方法间的层次关系应当如下：

![init-hierarchy](http://ww1.sinaimg.cn/large/6ad06ebbgw1ehjgol9vfgj20om0grgnp.jpg)

Swift 中还提供了 `@lazy` 修饰符，被它修饰的变量在第一次使用时才会执行初始化。

#### 闭包

语法什么的不写了。在 Swift 中，闭包也是 ARC 对象，它遵守一切 ARC 的规则。下面这段代码：

``` swift
var onTemperatureChange: (Int) -> Void = {}
func logTemperatureDifferences(initial: Int) {
  var prev = initial
  onTemperatureChange = { next in
    println("Changed \(next - prev)°F")
    prev = next
  }
}  // scope ends
```

的内存结构为：

![onTemperatureChange](http://ww1.sinaimg.cn/large/6ad06ebbgw1ehjhx76brhj206r07z3yk.jpg)

另外，函数也是 ARC 对象，下面这段代码：

``` swift
var onTemperatureChange: (Int) -> Void = {}
func logTemperatureDifferences(initial: Int) {
  var prev = initial
  func log(next: Int) {
    println("Changed \(next - prev)°F")
    prev = next
  }
  onTemperatureChange = log
}  // scope ends
```

的内存结构也是：

![onTemperatureChange-function](http://ww1.sinaimg.cn/large/6ad06ebbgw1ehjhx76brhj206r07z3yk.jpg)

但是这就会有个问题，当在闭包内部引用 `self` 时会产生循环造成内存泄露：

``` swift
class TemperatureNotifier {
  var onChange: (Int) -> Void = {}
  var currentTemp = 72
  init() {
    onChange = { temp in
      self.currentTemp = temp
    }
  }
}
```

![closure-self](http://ww4.sinaimg.cn/large/6ad06ebbgw1ehji03m8x8j20790a4aab.jpg)

可以想到的解决方法是用 `unowned` 来处理：

``` swift
class TemperatureNotifier {
  var onChange: (Int) -> Void = {}
  var currentTemp = 72
  init() {
    unowned let uSelf = self
    onChange = { temp in
      uSelf.currentTemp = temp
    }
  }
}
```

![closure-self-unowned](http://ww3.sinaimg.cn/large/6ad06ebbgw1ehjj9k50ymj206x0a074k.jpg)

其实 Swift 提供了更简洁的写法，即用 `[unowned self]` 这种语法结构：

``` swift
class TemperatureNotifier {
  var onChange: (Int) -> Void = {}
  var currentTemp = 72
  init() {
    onChange = {[unowned self] temp in
      self.currentTemp = temp
    }
  }
}
```

#### 模式匹配

Swift 的模式匹配支持各种体位，可以这样：

``` swift
enum TrainStatus {
  case OnTime
  case Delayed(Int)
}

switch trainStatus {
  case .OnTime:
    println("on time")
  case .Delayed:
    println("delayed")
```

这样：

``` swift
switch trainStatus {
  case .OnTime:
    println("on time")
  case .Delayed(let minutes):
    println("delayed by \(minutes) minutes")
```

和这样：

``` swift
switch trainStatus {
  case .OnTime:
    println("on time")
  case .Delayed(1):
    println("nearly on time")
  case .Delayed(2...10):
    println("almost on time, I swear")
  case .Delayed(_):
    println("it'll get here when it's ready")
```

甚至这样：

``` swift
enum VacationStatus {
  case Traveling(TrainStatus)
  case Relaxing(daysLeft: Int)
}

switch vacationStatus {
  case .Traveling(.OnTime):
    tweet("Train's on time! Can't wait to get there!")
  case .Traveling(.Delayed(1...15)):
    tweet("Train is delayed.")
  case .Traveling(.Delayed(_)):
    tweet("OMG when will this train ride end #railfail")
```

匹配类型也是可以的：

``` swift
func tuneUp(car: Car) {
  switch car {
    case let formulaOne as FormulaOne:
      formulaOne.enterPit()
    case let raceCar as RaceCar:
      if raceCar.hasTurbo { raceCar.tuneTurbo() }
      fallthrough
    default:
      car.checkOil()
      car.pumpTires()
  }
}
```

匹配元组就更不用说了：

``` swift
let color = (1.0, 1.0, 1.0, 1.0)
switch color {
  case (0.0, 0.5...1.0, let blue, _):
    println("Green and \(blue * 100)% blue")
```

还可以加上 `where` 从句来对匹配结果做进一步约束：

``` swift
switch color {
  case let (r, g, b, 1.0) where r == g && g == b:
    println("Opaque grey \(r * 100)%")
```

来看一个综合运用的例子：

``` swift
func stateFromPlist(list: Dictionary) -> State? {
  switch (list["name"], list["population"], list["abbr"]) {
    case (.Some(let listName as NSString),
          .Some(let pop as NSNumber),
          .Some(let abbr as NSString))
    where abbr.length == 2:
      return State(name: listName, population: pop, abbr: abbr)
    default:
      return nil
  }
}
```

本例中通过运用模式匹配优雅地解决了 `AnyObject` 类型转换的问题，遇到类型错误会直接静默失败。

### Session 404 Advanced Swift

这个 Session 通过构建一个文字游戏来展示了 Swift 中稍深入些的知识。

#### 特殊的协议

* `LogicValue`, `if logicValue {`
* `Printable`, `"\(printable)"`
* `Sequence`, `for x in sequence`
* `IntegerLiteralConvertible`, `65536`
* `FloatLiteralConvertible`, `1.0`
* `StringLiteralConvertible`, `"abc"`
* `ArrayLiteralConvertible`, `[ a, b, c ]`
* `DictionaryLiteralConvertible`, `[ a: x, b: y ]`

#### 定义运算符

就像下面这样：

``` swift
operator infix ~ {}
func ~ (decorator: (Thing) -> String, object: Thing) -> String {
  return decorator(object)
}
```

在定义一个新运算符时要先声明它的前中后缀性。

#### Subscript

就像下面这样：

``` swift
extension Place {
  subscript(direction: Direction) -> Place? {
    get {
      return exits[direction]
    }
    set(destination: Place?) {
      exits[direction] = destination
    }
  }
}
```

然后就可以用 `place['somewhere']` 这种语法了。

#### 泛型和协议

泛型支持看起来很平凡，似乎只能做 `<T: Equatable>` 这种的约束，但是人家运行快……

为了更好地和泛型配合，Swift 的协议支持 `Self` 这种类型，在某类型实现协议时会直接匹配成该类型：

``` swift
protocol Equatable {
  func == (lhs: Self, rhs: Self) -> Bool
}

struct Temperature : Equatable {
  let value: Int = 0
}

func == (lhs: Temperature, rhs: Temperature) -> Bool {
  return lhs.value == rhs.value
}
```

#### 函数式编程

这个 Session 给出了一个记忆化函数的实现，以递归计算阶乘为例：

``` swift
func memoize( body: ((T)->U, T)->U ) -> (T)->U {
  var memo = DictionaryT, U>()
  var result: ((T)->U)!
  result = { x in
    if let q = memo[x] { return q }
    let r = body(result, x)
    memo[x] = r
    return r
  }
  return result
}

let factorial = memoize { factorial, x in x == 0 ? 1 : x * factorial(x - 1) }
```

分清这三个 `factorial` 分别是什么就很好理解了。

#### Swfit 编译器结构

![objc-swift](http://ww1.sinaimg.cn/large/6ad06ebbgw1ehjv9jgddrj20wg0frgo0.jpg)

