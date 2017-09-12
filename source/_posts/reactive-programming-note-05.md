---
title: 响应式编程笔记 05
date: 2014-06-16
url: reactive-programming-note-05
---

### 扩展例子：离散事件模拟

这个例子用来展示赋值和高阶函数是如何以一种有趣的方式组合起来的。在本例中将会构建一个数字电路模拟器，而这个模拟器基于一个通用的离散事件模拟框架。

数字电路由电线和功能模块组成，电线传递由功能模块转化过的信号。本例中用布尔值 `true` 和 `false` 来表示信号，基础的功能模块（门）有：

* 非门，输出为输入的反转
* 与门，输出为输入的合取
* 或门，输出为输入的析取

<!-- more -->

其他的功能模块由这些基础模块组合而成。这些模块都有一个反应时间（或者*延迟*），就是说它们的输出不会立刻因输入的改变而改变。

在 Scala 中，用类 `Wire` 表示电线：

``` scala
val a = new Wire; val b = new Wire; val c = new Wire
// or, equivalently:
val a, b, c = new Wire
```

用下面几个函数来表示门电路。每个函数都有副作用以创建门电路：

``` scala
def inverter(input: Wire, output: Wire): Unit
def andGate(a1: Wire, a2: Wire, output: Wire): Unit
def orGate(o1: Wire, o2: Wire, output: Wire): Unit
```

更复杂的模块可以由它们组成：

![halfAdder](http://ww4.sinaimg.cn/large/6ad06ebbgw1ehh2yf1cikj20a405ut8v.jpg)

``` scala
def halfAdder(a: Wire, b: Wire, s: Wire, c:Wire): Unit = {
  val d = new Wire
  val e = new Wire
  orGate(a, b, d)
  andGate(a, b, c)
  inverter(c, e)
  andGate(d, e, s)
}
```

![fullAdder](http://ww2.sinaimg.cn/large/6ad06ebbgw1ehh2zsfnf6j20cz071q39.jpg)

``` scala
def fullAdder(a: Wire, b: Wire, cin: Wire, sum: Wire, cout: Wire): Unit = {
  val s = new Wire
  val c1 = new Wire
  val c2 = new Wire
  halfAdder(b, cin, s, c1)
  halfAdder(a, s, sum, c2)
  orGate(c1, c2, cout)
}
```

练习：下面的程序描述了什么逻辑功能？

``` scala
def f(a: Wire, b: Wire, c: Wire): Unit = {
  val d, e, f, g = new Wire
  inverter(a, d)
  inverter(b, e)
  andGate(a, e, f)
  andGate(b, d, g)
}
```

答案是 `a != b`。

下面来给出类 `Wire` 和这些函数的实现，它们是基于离散事件模拟 API 的。

一个离散事件模拟器执行由用户在给定*时刻（moment）*提供的*动作（actions）*。一个动作如下定义：

``` scala
type Action = () => Unit
```

另外模拟器中的*时间（time）*是模拟的，和实际时间无关。

模拟器的 trait 具有以下签名：

``` scala
trait Simulation {
  /** 当前模拟时间 */
  def currentTime: Int = ???
  /** 在给定时间延迟后执行一段动作 */
  def afterDelay(delay: Int)(block: => Unit): Unit = ???
  /** 开始模拟直到没有在等待的动作 */
  def run(): Unit = ???
}
```

一个电线必须支持三种基本操作：

``` scala
getSignal: Boolean            // 返回当前线路上的信号
setSignal(sig: Boolean): Unit // 修改线路上的信号
addAction(a: Action): Unit    // 在线路上附加指定的动作，在线路信号改变时这些动作都会被执行```

类 `Wire` 的实现如下：

``` scala
class Wire {
  private var sigVal = false
  private var actions = List[Action] = Nil
  def getSignal: Boolean = sigVal
  def setSignal(s: Boolean): Unit =
    if (s != sigVal) {
      sigVal = s
      actions foreach (_())
    }
  def addAction(a: Action): Unit = {
    actions = a :: actions
    a()
  }
}
```

一个电线的状态有两个私有变量来表示，`sigVal` 表示当前信号值，`actions` 表示电线上附加的动作。

那三个门电路可以用以下方法实现，注意每个变化都在一定延迟之后才生效：

``` scala
def inverter(input: Wire, output: Wire): Unit = {
  def invertAction(): Unit = {
    val inputSig = input.getSignal
    afterDelay(InverterDelay) { output setSignal !inputSig }
  }
  input addAction invertAction
}
def andGate(in1: Wire, in2: Wire, output: Wire): Unit = {
  def andAction(): Unit = {
    val in1Sig = in1.getSignal
    val in2Sig = in2.getSignal
    afterDelay(AndGateDelay) { output setSignal (in1Sig & in2Sig) }
  }
  in1 addAction andAction
  in2 addAction andAction
}
def orGate(in1: Wire, in2: Wire, output: Wire): Unit = {
  def orAction(): Unit = {
    val in1Sig = in1.getSignal
    val in2Sig = in2.getSignal
    afterDelay(OrGateDelay) { output setSignal (in1Sig | in2Sig) }
  }
  in1 addAction orAction
  in2 addAction orAction
}
```

练习：如果在 `afterDelay` 中直接计算 `in1Sig` 和 `in2Sig`，那么功能还相同吗？

``` scala
def orGate2(in1: Wire, in2: Wire, output: Wire): Unit = {
  def orAction(): Unit = {
    afterDelay(OrGateDelay) { output setSignal (in1.getSignal | in2.getSignal) }
  }
  in1 addAction orAction
  in2 addAction orAction
}
```

肯定不同啦，因为这个代码块在一段延迟后才调用，计算的信号值是延迟之后的。

接下来实现 `Simulation`，方法是用一个*日程表（agenda）*来保存所有的*事件（events）*，每个事件保存着动作的内容和开始时间。

``` scala
abstract class Simulation {
  type Action = () => Unit
  case class Event(time: Int, action: Action)
  private type Agenda = List[Event]
  private var agenda: Agenda = List()
  private var curtime = 0
  def currentTime: Int = curtime
  def afterDelay(delay: Int)(block: => Unit): Unit = {
    val item = Event(currentTime + delay, () => block)
    agenda = insert(agenda, item)
  }
  private def insert(ag: List[Event], item: Event): List[Event] = ag match {
    case first :: rest if first.time 
      first :: insert(rest, item)
    case _ =>
      item :: ag
  }
  private def loop(): Unit = agenda match {
    case first :: rest =>
      agenda = rest
      curtime = first.time
      first.action()
      loop()
    case Nil =>
  }
  def run(): Unit = {
    afterDelay(0) {
      println("*** simulation started, time = " + currentTime + " ***")
    }
    loop()
  }
}
```

在开始模拟之前，需要有一种办法来查看线路上的信号，因此定义函数 `probe`：

``` scala
def probe(name: String, wire: Wire): Unit = {
  def probeAction(): Unit = {
    println(s"$name $currentTime value = ${wire.getSignal}")
  }
  wire addAction probeAction
}
```

将延迟常量封装在自己的 trait 里会更方便些：

``` scala
trait Parameters {
  def InverterDelay = 2
  def AndGateDelay = 3
  def OrGateDelay = 5
}
```

至此模拟器全部实现完成了。

练习：若用以下方法定义或门（使用与门和非门），在包含了这个或门的电路中模拟结果会有变化吗？

![orGateAlt](http://ww3.sinaimg.cn/large/6ad06ebbgw1ehh31cbgvaj208v02ma9y.jpg)

``` scala
def orGateAlt(in1: Wire, in2: Wire, output: Wire): Unit = {
  val notIn1, notIn2, notOut = new Wire
  inverter(in1, notIn1); inverter(in2, notIn2)
  andGate(notIn1, notIn2, notOut)
  inverter(notOut, output)
}
```

答案是时间会变化，另外可能会产生多余的事件。原因是当这个或门的输入发生变化时，变化会依次传递到其内的各个门电路中，最终导致多余事件产生。如下图所示：

![orGateAlt-effect](http://ww3.sinaimg.cn/large/6ad06ebbgw1ehh31sq7y2j20a004874d.jpg)

由本例可以看出，状态和赋值的引入使得计算模型更复杂了，尤其是失去了引用透明特性。但另一方面，赋值可以更优美地构建一些特定程序，比如本例中的离散事件模拟。

