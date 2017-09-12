---
title: 修正 blanket 主题
date: 2012-12-10
url: fix-blanket-theme
---

左挑右选，看中了[blanket](https://github.com/octopress-themes/blanket)这个[octopress](http://octopress.org/)主题。整体很简洁，但估计是作者不细心，这主题有瑕疵：

<!-- more -->

* 边栏开关没对齐
    ![blanket-theme-sidebar-toggle](http://ww4.sinaimg.cn/large/a74eed94jw1dznzthdxh3j.jpg)
* 标题周围有多余边框
    ![blanket-theme-border](http://ww3.sinaimg.cn/large/a74e55b4jw1dznzu3dm08j.jpg)

本来不想管就这样忍了，但看了两天实在是很不爽。作为一个程序猿，自己动手，丰衣足食。

第一个问题，看了octopress自带的classic主题，发现两个代码基本一致。只是classic的`<div>`和背景同色，所以看不出来：

![classic-theme-sidebar-toggle](http://ww3.sinaimg.cn/large/a74ecc4cjw1dznzsv7rlzj.jpg)

恩，把`<div>`向上提就好了~在`sass/base/_layout.scss`文件中，找到`padding-top: $pad-medium/2;`这句注释掉即可，一共有两处。

第二个问题，是blanket加在`<article>`上的边框，用来显示周围那个小灰线。作者没好好测试嘛，在Archive页面的文章标题上就多余了。

我是这样改的：在文件`sass/partials/_archive.scss`中，给`#blog-archives article`加上`border: 0;`，在第5行左右。

``` sass
#blog-archives {
  article {
    border: 0;
```

Ok，搞定。Github上我改好的：[stormluke/blanket](https://github.com/stormluke/blanket)。

