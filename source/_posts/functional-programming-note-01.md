---
title: 函数式编程笔记 01
date: 2013-06-25
url: functional-programming-note-01
---

Cousera 上 [Functional Programming Prinples in Scala](https://class.coursera.org/progfun-002/class) 的笔记。

### 编程范式

范式描述了某些科学学科中独特的概念或者思考模式。

主要的编程范式：

* 命令式编程
* 函数式编程
* 逻辑式编程

与它正交：

* 面向对象编程

<!-- more -->

### 命令式编程

* 修改可变变量
* 赋值
* 诸如if-then-else、loops、break、continue、return等的控制结构

简单来理解，命令式编程就是Von Neumann计算机上的指令序列。

### 命令式的程序和计算机

* 可变变量 ≈ 内存单元
* 变量解引用 ≈ load指令
* 变量赋值 ≈ store指令
* 控制结构 ≈ jump指令

这在程序规模变大时会出现问题。怎样才能避免逐字的翻译式的编程？

John Backus 1978年在Turing Award上的讲稿：[*Can Programming be Liberated from the von. Neumann Style?*](http://www.thocp.net/biographies/papers/backus_turingaward_lecture.pdf)

Jone Backus是第一个高级语言Fortran的发明者。

### 规模增大

纯命令式编程被Von Neumann所限制：

> *“One tends to conceptualize data structures word-by-word”*

需要其他的方式来定义类似于集合、多项式、几何图形、串、文档等这些高级的抽象。

理想的方法是提出集合、形状、串等的定理。

### 什么是定理

定理包括

* 一个或多个数据类型
* 这些类型上的运算符
* 值和运算符之间关系的规则

定理并不描述变化！

### 定理不包括变化

一个定义两个多项式的和的例子：

``` scala
(a * x + b) + (c * x + d) = (x + c) * x + (b + d)
```

它并没有定义一个在改变系数的同时保持多项式相等的运算符！

但是在命令式编程中却可以这样写：

``` scala
class Polynomial { double[] coefficient; }
Polynomial p = …;
p.coefficient[0] = 42;
```

（系数`coefficient[0]`变了但多项式`p`没变）

另外一个例子是字符串中的`++`运算符：

``` scala
(a ++ b) ++ c = a ++ (b ++ c)
```

它并没有定义一个在改变序列元素的同时保持序列相等的运算符！

Java在这儿做的不错……它的String是不可变的。

### 对编程的影响

如果想根据它们的数学定理来实现高级概念，那就没有变化的地方。

* 数学不承认它
* 变化会毁掉定理中有用的规律

因此

* 用函数来表达运算符的定理
* 避免变化
* 有了一个强大的方法来抽象和构造函数

### 函数式编程

* 在狭义上，函数式编程意味着没有可变变量，赋值，循环和其他的命令式控制结构
* 在广义上，函数式编程意味着专注于函数
* 特别是，函数可以是能被产生、消耗和构造的值
* 这在函数式语言中都特别简单

### 函数式编程语言

函数是一等公民。

* 在任何地方都可以被定义，包括在另一个函数内部
* 像其他值一样，可以作为参数传递给其他函数，也可以作为值返回
* 像其他值一样，有一套构造函数的运算符

狭义上的函数式编程语言：

* Pure Lisp, XSLT, XPath, XQuery, FP
* Haskell (without I/O Monad or UnsafPerformIO)

广义上的函数式编程语言：

* Lisp, Scheme, Racket, Clojure
* SML, Ocaml, F#
* Haskell (full language)
* Scala
* Smalltalk, Ruby

### 函数式编程语言历史

* 1959 - Lisp
* 1975-77 - ML, FP, Scheme
* 1978 - Smalltalk
* 1986 - Standard ML
* 1990 - Haskell, Erlang
* 1999 - XSLT
* 2000 - OCaml
* 2003 - scala, XQuery
* 2005 - F#
* 2007 - Clojure

### 资源

[Leaning Resources](https://class.coursera.org/progfun-002/wiki/view?page=LearningResources)

