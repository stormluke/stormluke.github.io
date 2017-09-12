---
title: Java 拾遗 02 - Java 7 的异常处理
date: 2016-08-03 20:00
url: about-java-02
---

### Multicatch

Java 7 新加了 multicatch 特性，一个 `catch` 语句中可以捕获多种异常：

``` java
try {
  String fileText = getFile(fileName);
  cfg = verifyConfig(parseConfig(fileText));
} catch (FileNotFoundException | ParseException | ConfigurationException e) {
  System.err.println("Config file '" + fileName + "' is missing or malformed");
}
```

<!-- more -->

### final 重抛

Java 7 之前重抛异常时会被强制限制为 `catch` 到的异常类型，例如：

``` java
try {
  doSomethingWhichMightThrowIOException();
  doSomethingElseWhichMightThrowSQLException();
} catch (Exception e) {
  throw e;
}
```

这段代码中的 `e` 可能为 `IOException` 或者 `SQLException` 类型，但是真实的类型被覆盖了。Java 7 中可以用 `final` 来修饰异常类型，这样重抛时的类型不会被改变：

``` java
try {
  doSomethingWhichMightThrowIOException();
  doSomethingElseWhichMightThrowSQLException();
} catch (final Exception e) {
  throw e;
}
```

### try-with-resources

Java 中关闭资源的时候很容易出错，比如关闭一个流要记得先检查 `null`，再 `try` `close()`，出现异常还要记得处理（一般什么也做不了）等等。Java 7 中加入了自动管理资源的特性，如下：

``` java
try (OutputStream out = new FileOutputStream(file);
     InputStream is = url.openStream()) {
  byte[] buf = new byte[4096];
  int len;
  while ((len = is.read(buf)) > 0) {
    out.write(buf, 0, len);
  }
}
```

这段代码结束后会自动关闭资源。但是要小心，这样写是不对的：

``` java
try (ObjectInputStream in = new ObjectInputStream(new FileInputStream("someFile.bin"))) {
  ...
}
```

内部的 `new FileInputStream("someFile.bin")` 如果失败并不会被关闭，正确的方法是为每个资源声明独立变量。

TWR 还有个好处是改善了异常堆栈，比如说会抑制异常堆栈中的 `NullPointerException` 等等。

TWR 依靠新接口 [`AutoCloseable`](https://docs.oracle.com/javase/8/docs/api/java/lang/AutoCloseable.html) 实现，Java 7 大部分的资源类都实现了这个接口。

