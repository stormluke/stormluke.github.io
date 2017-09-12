---
title: Java 拾遗 01 - Colleciton 基础
date: 2016-08-03 08:00
url: about-java-01
---

在 [Oracle 的文档](https://docs.oracle.com/javase/tutorial/collections/index.html)中提到，Collection 是：

* 接口
* 实现
* 算法

<!-- more -->

的集合。Collection 的好处有：

* 减少编程工作量
* 提高程序速度和质量
* 允许无关的API之间的互操作性
* 让学习和使用新 API 更省力
* 让设计新 API 更省力
* 促进软件重用

Java 中的基础 Collection 接口有：

* `Set` / `SortedSet`
* `List`
    * `Queue` / `Deque`（双端队列，Double Ended QUEue）
* `Map` / `SortedMap`

Java 对这些接口提供了一些通用的实现：

* `Set`
    * `HashSet`，Java 1.2，无序，散列
    * `TreeSet`，Java 1.2，升序，红黑树
    * `LinkedHashSet`，Java 1.4，插入序，散列
* `List`
    * `ArrayList`，Java 1.2，数组
    * `LinkedList`，Java 1.2，链表
    * `ArrayDeque`，Java 1.6，数组
* `Map`
    * `HashMap`，Java 1.2，无序，散列
    * `TreeMap`，Java 1.2，升序，红黑树
    * `LinkedHashMap`，Java 1.4，插入序，散列

它们之间的关系如下：

![](http://ww3.sinaimg.cn/large/6ad06ebbgw1f6ghss8lpvj20z00zejw5.jpg)

值得注意的是 `Map` 不遵循 `Collection`，`Set` 的实现依赖于 `Map`。这些 Collection 均没有考虑线程安全。

某些面试题中问到 `Vector` 和 `ArrayList` 的区别，主要有以下几点：

* `Vector` 在 Java 1.0 加入，`ArrayList` 在 Java 1.2 加入
* `Vector` 线程安全（由 `synchronized` 实现），`ArrayList` 没考虑线程安全，但有专门的易互转的 `CopyOnWriteArrayList` 等来应对多线程应用
* `Vector` 每次扩容一倍，`ArrayList` 扩容 50%
* 在[这个 benchmark](https://dzone.com/articles/java-collection-performance) 中 `Vector` 比 `ArrayList` 略慢

所以 `Vector` 一般存在于面试题中。同样，`Hashtable`、`Dictionary`、`BitSet`、`Stack`、`Properties` 这些类也都是遗留的。

关于不同 Collection 的算法实现效率，可以参考 [Big-O Cheat Sheet（Java 无关）](http://bigocheatsheet.com/)／[big-o-summary-for-java-collections-framework-implementations（Java 相关）](http://stackoverflow.com/a/559862)。

更多关于 Collection 的参考：

* [Collections Framework Overview](http://docs.oracle.com/javase/6/docs/technotes/guides/collections/overview.html)
* [Annotated Outline of Collections Framework](http://docs.oracle.com/javase/6/docs/technotes/guides/collections/reference.html)




