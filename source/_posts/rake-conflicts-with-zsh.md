---
title: rake 和 zsh 冲突
date: 2012-12-08 08:00
url: rake-conflicts-with-zsh
---

现在用的shell是[oh-my-zsh](https://github.com/robbyrussell/oh-my-zsh)，发现执行`rake new_post["newpost"]`时提示`zsh: no matches found: new_post[newpost]`，不能新建文章。转到bash下再执行相同命令却能成功，想到可能是zsh的问题。Google之，在octpress的issues里找到了[答案](https://github.com/imathis/octopress/issues/117#issuecomment-3707975)：zsh会转义`[]`。

<!-- more -->

解决的方法有：

* `alias rake="noglob rake"`，`noglob`用来取消转义。
* `rake "new_post[title]"`。
* `rake new_post\[title\]`。

