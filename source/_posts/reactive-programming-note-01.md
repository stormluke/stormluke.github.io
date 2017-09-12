---
title: 响应式编程笔记 01
date: 2014-05-02
url: reactive-programming-note-01
---

Coursera: [Principles of Reactive Programming](https://class.coursera.org/reactive-001).

### 变化……

| 项目 | 十年前 | 现在 |
| --- | --- | --- |
| 服务器节点 | 十几个 | 上千个 |
| 响应时间 | 几秒 | 几毫秒 |
| 维护当机时间 | 几小时 | 无 |
| 数据量 | GB | TB或PB |

<!-- more -->

### ……需要新架构

* 之前：受控的服务器和容器 (containers)
* 现在：响应式应用
    * 事件驱动 (event-driven)
    * 可扩展 (scalable)
    * 抵抗力（适应性）强 (resilient)
    * 反应灵敏 (responsive)

### 响应式

韦氏词典中 reactive 解释为

> readily responsive to a stimulus

* 对事件响应（事件驱动）
* 对负载响应（可扩展）
* 对错误响应（抵抗力强）
* 对用户响应（反应快）

### 事件驱动

传统上：系统由多个线程组成，线程间通过共享同步的状态通信。

* 强耦合，难组合

现在：系统由弱耦合的事件处理器（event handlers）组成。

* 事件可以被异步地处理，无阻塞

### 可扩展

一个应用是可扩展的是指它可以根据自身使用情况来扩大缩小。

* 纵向扩展 (scale up): 利用多核处理并行性
* 横向扩展 (scale out): 利用多个服务器节点

可扩展的重点：最小化共享可变状态

横向扩展的重点：位置透明、抵抗力强

### 抵抗力强

一个应用抵抗力强是指它可以快速地从错误中恢复。

错误可以是：

* 软件错误
* 硬件错误
* 通信错误

一般来说，抵抗性不能事后再添加；而必须成为初始设计的一部分。

需要这些：

* 弱耦合
* 对状态的强封装
* 普遍的监管者 (supervisor) 架构

### 反应快

一个应用反应快是指其向用户提供丰富及时的交互，即使在高负载和错误存在的情况下也不受影响。

### 回调

处理事件时经常使用回调，比如用Java中的观察者：

``` scala
class Counter implements ActionListener {
  private var count = 0
  button.addActionListener(this)
  def actionPerformed(e: ActionEvent): Unit = {
    count += 1
  }
}
```

这有几个问题：

* 需要共享可变状态
* 不能组合
* 很快会导致“回调地狱” (call-back hell)

### 怎样做的更好

使用函数式编程中基础的组件来完成可组合的事件抽象。

* 事件是第一等级 (Events are first class)
* 事件常表示为信息 (Events are often represented as messages)
* 事件的处理器也是第一等级 (Handlers of events are also first-class)
* 复杂的处理器可由简单的组合而成 (Complex handlers can be composed from primitive ones)

### 这门课的内容

* 回顾函数式编程
* 函数式范式中一个重要的类：*monads*
* 在包含状态环境下的函数式编程
* 事件上的抽象：*futures*
* 事件流上的抽象：*observables*
* 信息传递架构：*actors*
* 处理错误：*supervisors*
* 横向扩展：*distributed actors*


