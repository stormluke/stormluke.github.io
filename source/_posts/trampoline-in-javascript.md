---
title: 蹦床（trampoline）原理
date: 2015-12-22
url: trampoline-in-javascript
---

摘录自 *Functional JavaScript*。

来看一个相互递归的例子

``` js
function evenSteven(n) {
  if (n === 0)
    return true
  else
    return oddJohn(Math.abs(n) - 1);
}

function oddJohn(n) {
  if (n === 0)
    return false
  else
    return evenSteven(Math.abs(n) - 1);
}
```

两个函数相互回弹调用彼此，递减某个绝对值，直到一方到达零为止。这是一个相当优雅的解决方式。

<!-- more -->

``` js
evenSteven(4);
//=> true
oddJohn(11);
//=> true
```

尽管技术上可以实现尾递归的优化，但目前 JavaScript 引擎并不支持。因此在调用以上函数时可能会遇到这个错误：

``` js
evenSteven(100000);
// Uncaught RangeError: Maximum call stack size exceeded (or some variant)
```

这种错误称为“blowing the stack”，是因为 `evenSteven` 和 `oddJohn` 互相调用过多造成了栈溢出。

可以是用蹦床（trampoline）原理的控制结构来消除这类错误。它的基本原理是，使用蹦床展平调用，而不是深度嵌套的递归调用。

首先看看如何来手动修复这两个函数使得递归不会溢出。一个方法是返回一个函数，它包装调用，而不是直接调用。可以使用 `partial1` 来实现这一点：

``` js
function partial1(fun, arg1) {
  return fun.bind(undefined, arg1);
}

这时对应的函数为：

``` js
function evenOline(n) {
  if (n === 0)
    return true;
  else
    return partial1(oddOline, Math.abs(n) - 1);
}

function oddOline(n) {
  if (n === 0)
    return false;
  else
    return partial1(evenOline, Math.abs(n) - 1);
}
```

这两个函数返回一个包装函数而不是直接进行 `evenOline` 和 `oddOline` 的互相调用。调用终止情况时都能正常工作：

``` js
evenOline(0);
//=> true
oddOline(0);
//=> true
```

现在可以手动调用递归来展平：

``` js
oddOline(3);
//=> function() { return evenOline(Math.abs(n) - 1) }
oddOline(3)();
//=> function() { return oddOline(Math.abs(n) - 1) }
oddOline(3)()();
//=> function() { return evenOline(Math.abs(n) - 1) }
oddOline(3)()()();
//=> true
oddOline(200000001)()()()(); //... a bunch more ()s
```

看起来能用了，但可以提供另外一个函数 `trampoline`，从程序执行来进行扁平化处理：

``` js
function trampoline(fun /*, args */) {
  var result = fun.apply(fun, _.rest(arguments));
  while (_.isFunction(result)) {
    result = result();
  }
  return result;
}
```

`trampoline` 所做的是不断调用函数的返回值，直到它不再是一个函数。具体是这样工作的：

``` js
trampoline(oddOline, 3);
//=> true
trampoline(evenOline, 200000);
//=> true
trampoline(oddOline, 300000);
//=> false
trampoline(evenOline, 200000000);
// wait a few seconds
//=> true
```

由于调用链的间接性，使用蹦床增加了相互递归函数的一些开销。然而，慢总比溢出好。如果不想强迫用户使用 `trampoline`，只是为了避免堆栈溢出，其实也可以隐藏其外观：

``` js
function isEvenSafe(n) {
  if (n === 0)
    return true;
  else
    return trampoline(partial1(oddOline, Math.abs(n) - 1));
}

function isOddSafe(n) {
  if (n === 0)
    return false;
  else
    return trampoline(partial1(evenOline, Math.abs(n) - 1));
}
```

试试能否正常工作：

``` js
isOddSafe(2000001);
//=> true
idEvenSafe(2000001);
//=> false
```

