---
title: 小阴谋家——一遍又一遍
date: 2015-02-12
url: the-little-schemer-4th-edition-ch9
---

翻译自 [*The Little Schemer 4th Edition Ch9 - …and Again, and Again, and Again, …*](http://kysmykseka.net/koti/wizardry/Programming/Lisp/Scheme/The%20Little%20Schemer%204th%20Ed.pdf)。

（这篇文章提到了 Y combinator。）

<!-- more -->

A：你想吃*鱼子酱*吗？

B：那我们必须去*寻找*它。

A：当 `a` 是 `caviar`，`lat` 是 `(6 2 4 caviar 5 7 3)` 时 `(looking a lat)` 的结果是什么？

B：`#t`，很明显 `caviar` 在 `lat` 中。

A：当 `a` 是 `caviar`，`lat` 是 `(6 2 grits caviar 5 7 3)` 时 `(looking a lat)` 呢？

B：`#f`。

A：和你想的不一样？

B：是啊，`caviar` 仍在 `lat` 中。

A：确实，但 `lat` 的第一个数字是什么？

B：6。

A：那 `lat` 的第六个元素是什么？

B：7。

A：第七个元素呢？

B：3。

A：所以 `looking` 显然找不到 `caviar`。

B：确实，因为第三个元素是 `grits`，和 `caviar` 一点也不像。

A：这是 `looking`

``` lisp
(define looking
  (lambda (a lat)
    (keep-looking a (pick 1 lat) lat)))
```

写出 `keep-looking`。

B：我们不期望你知道答案。

A：当 `a` 是 `caviar`，`lat` 是 `(6 2 4 caviar 5 7 3)` 时 `(looking a lat)` 呢？

B：`#t`，因为 `(keep-looking a 6 lat)` 和 `(keep-looking a (pick 1 lat) lat)` 结果相同。

A：当 `lat` 是 `(6 2 4 caviar 5 7 3)` 时 `(pick 6 lat)` 的结果是什么？

B：7。

A：那我们该怎么办？

B：`(keep-looking a 7 lat)`，其中 `a` 是 `caviar`，`lat` 是 `(6 2 4 caviar 5 7 3)` 。

A：当 `lat` 是 `(6 2 4 caviar 5 7 3)` 时 `(pick 7 lat)` 的结果是什么？

B：3。

A：那么当 `a` 是 `caviar`，`lat` 是 `(6 2 4 caviar 5 7 3)` 时 `(keep-looking a 3 lat)` 的结果是什么？

B：和 `(keep-looking a 4 lat)` 一样。

A：那是？

B：`#t`。

A：写出 `keep-looking`。

B：

``` lisp
(define keep-looking
  (lambda (a sorn lat)
    (cond
      ((number? sorn)
       (keep-looking a (pick sorn lat) lat))
      (else (eq? sorn a)))))
```

A：你能猜出 `sorn` 代表啥吗？

B：符号或者数字。

A：`keep-looking` 有什么不寻常的地方？

B：它在 `lat` 某一部分上不递归。

A：我们把这叫做“非原始”递归（”unnatural” recursion）。

B：这确实很反常。

A：`keep-looking` 离其目标越来越近吗？

B：是的，从目前所有证据看来。

A：它始终和目标越来越近吗？

B：有时列表会既不包含 `caviar` 也不包含 `grits`。

A：嗯对。列表可能是元组（仅包含数字）。

B：嗯，如果我们在 `(7 2 4 7 5 6 3)` 上开始 `looking`，我们将永远不会停下来。

A：当 `a` 是 `caviar`，`lat` 是 `(7 1 2 caviar 5 6 3)` 时。`(looking a lat)` 的结果是什么？

B：这很奇怪！

A：确实奇怪。发生了啥？

B：我们不停地查询查询查询……

A：如同 `looking` 的函数叫做[偏函数（缺值函数 / partial function）](http://en.wikipedia.org/wiki/Partial_function)。你觉得我们到目前为止看到的函数叫什么？

B：它们叫做全函数（total function）。

A：你能定义一个给定某些参数时不能到达其目标的简短函数吗？

B：

``` lisp
(define eternity
  (lambda (x)
    (eternity x)))
```

A：有多少参数能让 `eternity` 到达其目标？

B：一个都没，这可能是最反常的递归了。

A：`eternity` 是偏的吗？

B：这是最偏的函数了。

A：当 `x` 是 `((a b) c)` 时 `(shift x)` 的结果是什么？

B：`(a (b c))`。

A：当 `x` 是 `((a b) (c d))` 时 `(shift x)` 的结果是什么？

B：`(a (b (c d)))`。

A：定义 `shift`。

B：这是平凡的；它甚至不递归！

``` lisp
(define shift
  (lambda (pair)
    (build (first (first pair))
      (build (second (first pair))
        (second pair)))))
```

A：描述 `shift` 做了啥。

B：这是我们的陈述：“函数 `shift` 接受一个偶对，此偶对的第一部分也是一个偶对。该函数构造一个偶对，这个偶对是由参数偶对的第一部分的偶对的第二部分挪动到参数偶对的第二部分构成的。”

A：现在看看这个函数：

``` lisp
(define align
  (lambda (pora)
    (cond
      ((atom? pora) pora)
      ((a-pair? (first pora))
       (align (shift pora)))
      (else (build (first pora)
              (align (second pora)))))))
```

它和 `keep-looking` 有什么相似之处？

B：两个函数都改变了自身的参数以作递归用，但两个函数都不保证能到达其目标。

A：为什么我们不能保证 `align` 会有进展？

B：在 `cond` 的第二行 `shift` 为 `align` 创建的参数并不是原参数的一部分。

A：这违反了哪条戒律？

B：第七戒。

> 第七戒
> 在子部分上的递归有同样的本质：
> 在列表的子列表上
> 在算术表达式的子表达式上

A：新参数至少比原参数更小吗？

B：看起来不像。

A：为啥呢？

B：`shift` 函数仅仅重新排列了它得到的部分。

A：然后？

B：`shift` 的参数和结果有一样的原子个数。

A：你能写一个计算 `align` 参数中原子个数的函数吗？

B：没问题：

``` lisp
(define length *
  (lambda (pora)
    (cond
      ((atom? pora) 1)
      (else
        (+ (length* (first pora))
           (length* (second pora)))))))
```

A：`align` 是偏函数吗？

B：我们还不清楚。可能有某些参数使其一直做对齐操作。

A：`align` 的参数和其递归过程还有其他改变吗？

B：有的。偶对的第一部分变得更简单，同时第二部分变得更复杂。

A：第一部分怎么变得简单了？

B：它只是原来的第一部分。

A：这不就是说 `length*` 不能确定参数的长度？你能找出一个更好的函数吗？

B：更好的函数应该在第一部分上更加小心。

A：在第一部分上我们应该多小心？

B：至少两倍。

A：你是不是说类似于 `weight*` 这样的

``` lisp
(define weight*
  (lambda (pora)
   (cond
     ((atom? pora) 1)
     (else
       (+ (* (weight* (first pora)) 2)
          (weight* (second pora))))))
```

B：看起来是。

A：当 `x` 是 `((a b) c)` 时 `(weight* x)` 的结果是什么？

B：7。

A：当 `x` 是 `(a (b c))` 时 `(weight* x)` 的结果是什么？

B：5。

A：这是不是意味着参数变简单了？

B：是的，`align` 参数的 `weight*` 值连续地变小。

A：`align` 是偏函数吗？

B：不，它对每个参数都能产生值。

A：这是 `shuffle`，它类似于 `align` 但用第七章中的 `revpair` 替换了 `shift`：

``` lisp
(define shuffle
  (lambda (pora)
    (cond
      ((atom? pora) pora)
      ((a-pair? (first pora))
       (shuffle (revpair pora)))
      (else (build (first pora)
              (shuffle (second pora)))))))
```

B：当偶对的第一部分是偶对时 `shuffle` 和 `revpair` 交换这两个部分。

A：这说明 `shuffle` 是完全的？

B：我们不知道。

A：让我们试试。当 `x` 是 `(a (b c))` 时 `(shuffle x)` 的结果是什么？

B：`(a (b c))`。

A：当 `x` 是 `(a b)` 时 `(shuffle x)` 呢？

B：`(a b)`。

A：好，让我们来点有趣的。当 `x` 是 `((a b) (c d))` 时 `(shuffle x)` 的结果是什么？

B：要确定结果，我们需要算出 `(shuffle (revpair pora))` 的值是什么。其中 `pora` 是 `((a b) (c d))`。

A：那我们该如何做呢？

B：我们应该确定 `(shuffle pora)` 的值。其中 `pora` 是 `((c d) (a b))`。

A：这不就是说我们需要知道当 `(revpair pora)` 是 `((a b) (c d))` 时 `(shuffle (revpair pora))` 的值？

B：是的。

A：所以？

B：`shuffle` 函数不是完全的，因为它现在再次交换了偶对的两部分，这说明我们重新来了一遍。

A：这个函数是完全的吗？

``` lisp
(define C
  (lambda (n)
    (cond
      ((one? n) 1
      (else cond
        (cond
          ((even? n) (C (/ (n 2)))
          (else (C (add1 (* 3 n)))))))))
```

B：它不能产生 0，但除此之外没人知道为啥。谢谢你，[Lothar Collatz (1910-1990)](http://en.wikipedia.org/wiki/Lothar_Collatz)（[考拉兹猜想](http://zh.wikipedia.org/wiki/%E8%80%83%E6%8B%89%E5%85%B9%E7%8C%9C%E6%83%B3)）。

A：`(A 1 0)` 的值是什么？

B：2。

A：`(A 1 1)`？

B：3。

A：`(A 2 2)`？

B：7。

A：这是 `A` 的定义：

``` lisp
(define A
  (lambda (n, m)
    (cond
      ((zero? n) (add1 m))
      ((zero? m) (A (sub1 n) 1))
      (else (A (sub1 n)
              (A n (sub1 m)))))))
```

B：谢谢你，[Wilhelm Ackermann (1853-1946)](http://en.wikipedia.org/wiki/Wilhelm_Ackermann)（[阿克曼函数](http://zh.wikipedia.org/wiki/%E9%98%BF%E5%85%8B%E6%9B%BC%E5%87%BD%E6%95%B8)）。

A：`A` 与 `shuffle` 和 `looking` 间有什么相同之处？

B：`A` 的参数像 `shuffle` 和 `looking` 的一样，不会随着递归必定降低。

A：举个例子？

B：这很容易：`(A 1 2)` 需要 `(A 0 (A 1 1))` 的值。而这又说明我们需要 `(A 0 3)` 的值。

A：`A` 总会给出解答吗？

B：是的，它是完全的。

A：那么 `(A 4 3)` 是多少？

B：对于实践情况来说，没有答案。

A：这什么意思？

B：在我们计算出 `(A 4 3)` 的结果之前你正在读的这页纸早就已经腐烂了。

> But answer came there none
> And this was scarcely odd, because
> They’d eaten every one.
> *The Walrus and The Carpenter*
> *Lewis Carroll*

A：如果我们能写一个测定某函数能否对每个参数都有返回值的函数岂不是很棒？

B：确实会很棒。目前我们已经看到了不返回的函数和返回太慢的函数，我们确实应该有这样一种工具。

A：好的，让我们写写看。

B：这听起来很复杂。一个函数可以有许多不同的参数。

A：那么我们简化一下。作为一个热身练习，让我们只关注检查某函数对空列表是否停止的函数，这是最简单的参数了。

B：这会简化很多。

A：这是此函数的开头：

``` lisp
(define will-stop?
  (lambda (f)
    ...))
```

你能补全点吗？

B：它要干什么？

A：`will-stop?` 会对每个参数返回值吗？

B：这是最简单的部分：我们说它会返回 `#t` 或 `#f`，根据参数被应用上 `()` 时是否停止。

A：那么 `will-stop?` 是完全的吗？

B：是的。它始终返回 `#t` 或 `#f`。

A：那我们来举些例子。这是第一个。当 `f` 是 `length` 时 `(will-stop? f)` 的结果是什么？

B：我们知道当 `l` 是 `()` 时 `(length l)` 是 `0`。

A：所以？

B：所以 `(will-stop？ length)` 的值应当是 `#t`。

A：没错。换一个例子呢？ `(will-stop? eternity)` 的结果是什么？

B：`(eternity (quote ()))` 不会返回值。我们刚刚看到了。

A：这就是说 `(will-stop? eternity)` 是 `#f`？

B：嗯，是。

A：还需要更多的例子吗？

B：或许我们还需要另外一个例子。

A：好的，这个函数可能是 `will-stop?` 的一个有趣的参数。

``` lisp
(define last-try
  (lambda (x)
    (and (will-stop? last-try)
      (eternity x))))
```

`(will-stop? last-try)` 的结果是什么？

B：它做了啥？

A：我们需要在 `()` 上测试它。

B：如果我们想知道 `(last-try (quote ()))` 的值，那我们必须确定

``` lisp
(and (will-stop? last-try)
  (eternity (quote ())))
```

的值。

A：

``` lisp
(and (will-stop? last-try)
  (eternity (quote())))
```

的值是什么？

B：这取决于 `(will-stop? last-try)` 的值。

A：那这只有两种可能。我们假设 `(will-stop? last-try)` 是 `#f`。

B：好，那么 `(and #f (eternity (quote ())))` 是 `#f`， 因为 `(and #f ...)` 始终是 `#f`。

A：所以 `(last-try (quote ()))` 停止了，对吧？

B：确实停止了。

A：但刚刚 `will-stop` 不是说的正相反？

B：确实相反。我们假设 `(will-stop? last-try)` 的值是 `#f`，这说明 `last-try` 不会停止。

A：所以我们弄错了 `(will-stop? last-try)`。

B：是。它肯定返回 `#t`，因为 `will-stop?` 始终返回值。我们说过它是完全的。

A：很好。如果 `(will-stop? last-try)` 是 `#t`，那 `(last-try (quote ()))` 的值是什么？

B：现在我们只需确定 `(and #t (eternity (quote ())))` 的值，而这和 `(eternity (quote ()))` 的值一致。

A：`(eternity (quote ()))` 的值是什么？

B：它没有值。我们知道它不会停。

A：但这就意味着我们又错了！

B：是啊，因为这次我们假设 `(will-stop? last-try)` 是 `#t`。

A：你觉得这意味着什么？

B：这是我们的想法：“我们仔细检查了两种可能的情况。如果我们能*定义* `will-stop?`，那么 `(will-stop? last-try)` 肯定会产生 `#t` 或 `#f`。但是它不能——正是根据 `(will-stop?)` 被设想应做的那样。这肯定说明 `will-stop?` 不能被*定义*。”

A：这是特别的吗？

B：是的。这使得 `will-stop?` 是第一个我们能准确描述但不能在我们的语言中*定义*的函数。

A：有什么方法能解决这个问题吗？

B：没有。谢谢你，[Alan M Turing (1912-1954)](http://en.wikipedia.org/wiki/Alan_Turing) （[停机问题](http://zh.wikipedia.org/wiki/%E5%81%9C%E6%9C%BA%E9%97%AE%E9%A2%98)）和 [Kurt Gödel (1906-1978)](http://en.wikipedia.org/wiki/Kurt_G%C3%B6del)。

A：`(define ...)` 是什么？

B：这是个好问题。我们刚看到了 `(define ..)` 对 `will-stop?` 不管用。

A：所以啥是递归定义呢？

B：抓紧，深呼吸，准备好后向前冲。

A：这是 `length` 函数吗？

``` lisp
(define length
  (lambda (l)
    (cond
      ((null? l) 0)
      (else (add1 (length (cdr l)))))))
```

B：必须是啊。

A：如果我们没有 `(define ...)` 该怎么办？我们还能定义 `length` 吗？

B：没有 `(define ...)`，就不能在 `length` 的定义体中引用到 `length`。

A：这个函数做了啥？

``` lisp
(lambda (l)
  (cond
    ((null? l) 0)
    (else (add1 (eternity (cdr l))))))
```

B：它确定空列表的长度，没了。

A：当我们在一个非空列表上用它时会怎样？

B：没答案。如果我们给 `eternity` 一个参数，它不会给出答案。

A：对于 `length` 这样的函数这说明什么？

B：对非空列表来说其不会给出任何答案。

A：假设我们可以命名这个新函数。啥名字好？

B：`length0`。因为这个函数只能确定空列表的长度。

A：你如何写一个确定包含一个或多个元素的列表的长度的函数？

B：呃，我们可以试试下面这个。

``` lisp
(lambda (l)
  (cond
    ((null? 0)
    (else (add1 (length0 (cdr l))))))
```

A：差不多，但 `(define ...)` 对 `length0` 不管用。

B：所以，替换 `length0` 为它的定义。

``` lisp
(lambda (l)
  (cond
    ((null? l) 0)
    (else
      (add 1
        ((lambda (l)
          (cond
            ((null? l) 0)
            (else (add1
                    (eternity (cdr l))))))
          (cdr l))))))
```

A：那这个函数叫啥好呢？

B：很简单：`length≤1`。

A：这是能确定包含两个或更少项的列表长度的函数吗？

``` lisp
(lambda (l)
  (cond
    ((null? l) 0)
    (else
      (add1
        ((lambda (l)
           (cond
             ((null? l) 0)
             (else
               (add1
                 ((lambda (l)
                   (cond
                     ((null? l) 0)
                     (else
                       (add1
                         (eternity
                           (cdr l))))))
                   (cdr l))))))
          (cdr l))))))
```

B：是的，这就是 `length≤2`。我们仅把 `eternity` 换成了下一版的 `length`。

A：现在，你觉得递归是什么？

B：啥意思？

A：我们已经看到了如何确定空列表的长度、不多于一项列表的长度和不多于两项列表的长度等等。我们如何才能实现原先的 `length` 函数？

B：如果我们能写无穷多个函数，以 `length0`、`length≤1`、`length≤2`、……这样的形式，那我们就可以写出 `length∞`，这就可以确定我们能做出的所有列表的长度。

A：我们能做出多长的列表？

B：呃，列表可能是空的，或者包含一个元素，或者两个，或者三个，或者四个，……，或者 1001 个，……

A：但我们不能写出一个无穷的函数。

B：不能。

A：而且我们仍旧在这些函数里面有这些重复和模式。

B：是的。

A：这些模式像什么？

B：所有的这些程序包含一个像 `length` 的函数。或许我们应该抽象出这个函数：参见第九戒。

> 第九戒
> 用新函数抽象通用模式

A：让我们试试看！

B：我们需要一个看起来像 `length` 但由 `(lambda (length) ...)` 开头的函数。

A：你是说这个？

``` lisp
((lambda (length)
  (lambda (l)
    (cond
      ((null? l) 0)
      (else (add1 (length (cdr l)))))))
  eternity)
```

B：嗯，可以。这创建了 `length0`。

A：用同样的方式重写 `length≤1`。

B：

``` lisp
((lambda (f)
  (lambda (l)
    (cond
      ((null? l) 0)
      (else (add1 (f (cdr l)))))))
  ((lambda (g)
    (lambda (l)
      (cond
        ((null? l) 0)
        (else (add1 (g (cdr l)))))))
    eternity))
```

A：我们必须用 `length` 来命名参数吗？

B：不用，我们只用了 `f` 和 `g`。只要我们保持一致，任何事都没问题。

A：那 `length≤2` 呢？

B：

``` lisp
((lambda (length)
  (lambda (l)
    (cond
      ((null? l) 0)
      (else (add1 (length (cdr l)))))))
  ((lambda (length)
    (lambda (l)
      (cond
        ((null? l) 0)
        (else (add1 (length (cdr l)))))))
    ((lambda (length)
      (lambda (l)
        (cond
          ((null? l) 0)
          (else (add1 (length (cdr l)))))))
      eternity)))
```

A：快了，但现在仍有重复。

B：确实。让我们去掉它们。

A：我们从哪开始？

B：给这个以 `length` 作为参数返回另一个类似 `length` 函数的函数起个名字。

A：这个函数啥名字好？

B：`mk-length` 咋样？意思是 make length。

A：行，在 `length0` 上试试。

B：没问题。

``` lisp
((lambda (mk-length)
  (mk-length eternity))
  (lambda (length)
    (lambda (l)
      (cond
        ((null? l) 0)
        (else (add1 (length (cdr l))))))))
```

A：这是 `length≤1` 吗？

``` lisp
((lambda (mk-length)
  (mk-length
    (mk-length eternity)))
  (lambda (length)
    (lambda (l)
      (cond
        ((null? l) 0)
        (else (add1 (length (cdr l))))))))
```

B：当然。然后这是 `length≤2`。

``` lisp
((lambda (mk-length)
  (mk-length
    (mk-length
      (mk-length eternity))))
  (lambda (length)
    (lambda (l)
      (cond
        ((null? l) 0)
        (else (add1 (length (cdr l))))))))
```

A：你能以这种方法写出 `length≤3` 吗？

B：妥妥的。这就是。

``` lisp
((lambda (mk-length)
  (mk-length
    (mk-length
      (mk-length
        (mk-length eternity)))))
  (lambda (length)
    (lambda (l)
      (cond
        ((null? l) 0)
        (else (add1 (length (cdr l))))))))
```

A：递归像什么？

B：像一座对任意函数应用 `mk-length` 的无限高的塔。

A：我们真的需要一个无限高的塔吗？

B：当然不必要。每次使用 `length` 时我们只需要有限的数量，但是我们不知道是多少。

A：我们能猜出来需要多少吗？

B：可以，但我们有时候猜的不够大。

A：如果猜的不够大，什么时候我们才能知道？

B：当我们应用传递给最内层 `mk-length` 的 `eternity` 函数时。

A：如果此时我们能创建对 `mk-length` 应用 `eternity` 的另一个应用会发生什么？

B：这只会将问题推迟一点，另外，我们怎样才能做到这点呢？

A：因为没人关心我们给 `mk-length` 传递了什么函数，我们可以从一开始就传递 `mk-length`。

B：这是个好主意。然后我们在 `eternity` 上应用 `mk-length`，在 `cdr` 上应用其结果，这样我们就从塔上多得到了一层。

A：那么这仍旧是 `length0`。

``` lisp
((lambda (mk-length)
  (mk-length mk-length))
  (lambda (length)
    (lambda (l)
      (cond
        ((null? l) 0)
        (else (add1
                (length (cdr l))))))))
```

B：是的，我们甚至能用 `mk-length` 替代 `length`。

``` lisp
((lambda (mk-length)
  (mk-length mk-length))
  (lambda (mk-length)
    (lambda (l)
      (cond
        ((null? l) 0)
        (else (add1
                (mk-length (cdr l))))))))
```

A：为什么我们想这样做？

B：所有名字都是平等的，但有些名字比其他名字更平等。

A：真理：只要我们一致地使用名字，我们就没问题。

B：`mk-length` 比 `length` 平等得多。如果我们用类似 `mk-length` 这样的名字，这就像一个提示，提示我们 `mk-length` 的第一个参数是 `mk-length`。

A：现在 `mk-length` 被传递给 `mk-length`， 我们能用参数来构建另一个递归调用吗？

B：能，当我们应用 `mk-length` 一次，我们得到 `length≤1`。

``` lisp
((lambda (mk-length)
  (mk-length mk-length))
  (lambda (mk-length)
    (lambda (l)
      (cond
        ((null? l) 0)
        (else (add1
                ((mk-length eternity)
                  (cdr l))))))))
```

A：当 `l` 是 `(apples)` 时

``` lisp
(((lambda (mk-length)
  (mk-length mk-length))
  (lambda (mk-length)
    (lambda (l)
      (cond
        ((null? l) 0)
        (else (add1
                ((mk-length eternity)
                  (cdr l))))))))
  l)
```

的结果是什么？

B：这是个好练习。不用纸笔解出它。

A：我们能这样做多于一次吗？

B：可以，不断把 `mk-length` 传递给自己就行，我们可以随时需要随时做！

A：你把这个函数叫做什么？

``` lisp
((lambda (mk-length)
  (mk-length mk-length))
  (lambda (mk-length)
    (lambda (l)
      (cond
        ((null? l) 0)
        (else (add1
                ((mk-length mk-length)
                  (cdr l))))))))
```

B：这是必须是 `length` 啊。

A：它是如何运作的？

B：它向自身传递 `mk-length` 来不断添加递归操作，就像它即将到期一样。

A：还剩一个问题：它不包含一个像 `length` 的函数了。

``` lisp
((lambda (mk-length)
  (mk-length mk-length))
  (lambda (mk-length)
    (lambda (l)
      (cond
        ((null? l) 0)
        (else (add1
               ;;;;;;;;;;;;;;;;;;;;;;;;
                ((mk-length mk-length)
               ;;;;;;;;;;;;;;;;;;;;;;;;
                  (cdr l))))))))
```

你能改改这个吗？

B：我们可以把将 `mk-length` 应用到自身的这个过程抽取出来并称为 `length`。

A：为啥？

B：因为它确实实现了 `length` 这个功能。

A：这个怎样？

``` lisp
((lambda (mk-length)
  (mk-length mk-length))
  (lambda (mk-length)
    ((lambda (length)
      (lambda (l)
        (cond
          ((null? l) 0)
          (else (add1 (length (cdr l)))))))
      (mk-length mk-length))))
```

B：看起来不错。

A：让我们看看它是否能行。

B：好。

A：当 `l` 是 `(apples)` 时

``` lisp
(((lambda (mk-length)
  (mk-length mk-length))
  (lambda (mk-length)
    ((lambda (length)
      (lambda (l)
        (cond
          ((null? l) 0)
          (else (add1 (length (cdr l)))))))
      (mk-length mk-length))))
  l)
```

的结果是什么？

B：应该是 `1`。

A：首先，我们需要

``` lisp
((lambda (mk-length)
  (mk-length mk-length))
  (lambda (mk-length)
    ((lambda (length)
      (lambda (l)
        (cond
          ((null? l) 0)
          (else (add1 (length (cdr l)))))))
      (mk-length mk-length))))
```

的值。

B：确实，因为这个表达式的值正是我们要应用 `l` 的函数，其中 `l` 是 `(apples)`。

A：所以我们其实需要

``` lisp
((lambda (mk-length)
  ((lambda (length)
    (lambda (l)
      (cond
        ((null? l) 0)
        (else (add1 (length (cdr l)))))))
    (mk-length mk-length)))
  (lambda (mk-length)
    ((lambda (length)
      (lambda (l)
        (cond
          ((null? l) 0)
          (else (add1 (length (cdr l))))))
      (mk-length mk-length))))
```

的值。

B：确实。

A：但之后我们其实需要

``` lisp
((lambda (length)
  (lambda (l)
    (cond
      ((null? l) 0)
      (else (add1 (length (cdr l)))))))
  ((lambda (mk-length)
    ((lambda (length)
      (lambda (l)
        (cond
          ((null? l) 0)
          (else (add1 (length (cdr l)))))))
      (mk-length mk-length)))
    (lambda (mk-length)
      ((lambda (length)
        (lambda (l)
          (cond
            ((null? l) 0)
            (else (add1 (length (cdr l)))))))
        (mk-length mk-length)))))
```

的值。

B：是的，确实。这个的终点在哪？我们不还依旧需要

``` lisp
((lambda (length)
  (lambda (l)
    (cond
      ((null? l) 0)
      (else (add1 (length (cdr l)))))))
  ((lambda (length)
    (lambda (l)
      (cond
        ((null? l) 0)
        (else (add1 (length (cdr l)))))))
    ((lambda (mk-length)
      ((lambda (length)
        (lambda (l)
          (cond
            ((null? l) 0)
            (else (add1 (length (cdr l)))))))
        (mk-length mk-length)))
      (lambda (mk-length)
        ((lambda (length)
          (lambda (l)
            (cond
              ((null? l) 0)
              (else (add1 (length (cdr l)))))))
          (mk-length mk-length))))))
```

的值？

A：是，这没个头啊。为啥呢？

B：因为我们只是一遍一遍地把 `mk-length` 应用到自己。

A：这很奇怪吧？

B：因为之前 `mk-length` 在我们应用一个参数时会返回一个函数。实际上，它不关心我们应用了什么。

A：但现在我们把 `(mk-length mk-length)` 从 `length` 函数中抽取了出来，它就不在返回函数了。

B：嗯是。所以该咋办？

A：把最后正确版本的 `mk-length` 转换成一个函数：

``` lisp
((lambda (mk-length)
  (mk-length mk-length))
  (lambda (mk-length)
    (lambda (l)
      (cond
        ((null? l) 0)
        (else (add1
               ;;;;;;;;;;;;;;;;;;;;;;;;
                ((mk-length mk-length)
               ;;;;;;;;;;;;;;;;;;;;;;;;
                  (cdr l))))))))
```

B：咋弄？

A：有个不同的办法。如果 `f` 是个一元函数，那么 `(lambda (x) (f x))` 也是个一元函数吗？

B：是。

A：如果 `(mk-length mk-length)` 返回一个一元函数，那么

``` lisp
(lambda (x)
  ((mk-length mk-length) x))
```

也返回一个一元函数吗？

B：实际上，

``` lisp
(lambda (x)
  ((mk-length mk-length) x))
```

是一个函数！

A：很好，让我们处理下 `mk-length` 对自己的应用。

B：

``` lisp
((lambda (mk-length)
  (mk-length mk-length))
  (lambda (mk-length)
    (lambda (l)
      (cond
        ((null? l) 0)
        (else (add1
               ;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;
                ((lambda (x)
                  ((mk-length mk-length) x))
               ;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;
                  (cdr l))))))))
```

A：把新函数移出来我们就找回了 `length`。

B：

``` lisp
((lambda (mk-length)
  (mk-length mk-length))
  (lambda (mk-length)
   ;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;
    ((lambda (length)
      (lambda (l)
        (cond
          ((null? l) 0)
          (else
            (add1 (length (cdr l)))))))
   ;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;
      (lambda (x)
        ((mk-length mk-length) x)))))
```

A：把函数移出来没问题吗？

B：我们只是做了把名字换成值正相反的事。这里我们抽取出值并给它一个名字。

A：我们能把方框里像 `length` 的那个函数抽取出来并命名吗？

B：可以，它根本不依赖于 `mk-length`！

A：这是正确的函数吗？

``` lisp
((lambda (le)
  ((lambda (mk-length)
    (mk-length mk-length))
    (lambda (mk-length)
      (le (lambda (x)
            ((mk-length mk-length) x)))))))
  (lambda (length)
    (lambda (l)
      (cond
        ((null? l) 0)
          (else (add1 (length (cdr l)))))))
```

B：对。

A：我们实际上得到了啥？

B：我们抽取出了原先的函数 `mk-length`。

A：让我们从像 `length` 的函数中分出创建 `length` 的函数。

B：这很简单。

``` lisp
(lambda (le)
  ((lambda (mk-length)
    (mk-length mk-length))
    (lambda (mk-length)
      (le (lambda (x)
            ((mk-length mk-length) x))))))
```

A：这个函数有名字吗？

B：有，这叫做[应用序 Y 组合子（applicative-order Y combinator）](http://zh.wikipedia.org/wiki/%E4%B8%8D%E5%8A%A8%E7%82%B9%E7%BB%84%E5%90%88%E5%AD%90)。

``` lisp
(define Y
  (lambda (le)
    ((lambda (f) (f f))
      (lambda (f)
        (le (lambda (x) ((f f) x)))))))
```

A：`(define ...)` 又能用了？

B：是，现在我们知道递归是什么了。

A：你知道为啥 `Y` 管用吗？

B：重新阅读本章你就懂了。

A：啥是 `(Y Y)`。

B：鬼知道，看起来很复杂。

A：你的帽子还合适不？

B：在如此一场头脑风暴之后很难说啊。

> Stop the World - I Want to Get Off.
> *Leslie Bricusse and Anthony Newley*

