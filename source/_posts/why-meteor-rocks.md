---
title: Why Meteor Rocks!
date: 2013-06-06
url: why-meteor-rocks
---

这是 *Getting Started with Meteor.js JavaScript Framework* 一书第三章 *Why Meteor Rocks!* 的翻译。

### 为啥 Meteor 牛

Meteor是一种颠覆性（好的方面！）技术。它采用Model View View-Model（MVVM）设计模式，开启了一种新型web应用。

这一章解释了web应用是如何变化的、为什么重要以及Meteor是如何具体地通过MVVM实现了现代web应用。

<!-- more -->

在这章结束时，你会学到：

* 什么是现代web应用
* MVVM的含义，以及它为什么与众不同
* Meteor是怎样用MVVM来创建现代web应用的
* Meteor内部的模板——开始使用MVVM

#### 现代 web 应用

我们的世界正在改变。

随着显示技术、运算和存储能力的持续进步，几年前不可能的东西现在不仅变成了现实，而且成为一个优秀应用不可或缺的部分。尤其是Web经历了沧海桑田的变化。

##### Web 应用的起源（client / server）

刚开始时，web 服务器和客户端通过模仿**哑终端**的方式来进行交互。那时服务器拥有远超客户端的处理能力，因此承担处理数据（写记录到数据库，数学运算和文本查询等）、将数据转换成可读格式（把数据库记录转换成 HTML 等）和将结果发送到客户端以呈现给用户等等的任务。

就是说，服务器负责所有的工作，而客户端则更多扮演一个显示器或哑终端的角色。这个设计模式叫做……等一下……client / server 设计模式。

![client/server](http://ww2.sinaimg.cn/large/6ad06ebbjw1e5esjouamij20at07mmxe.jpg)

这个从六七十年代的哑终端和大型主机借鉴而来的设计模式，是我们所知的 Web 的开端，并且是我们看到 Internet 时所想到的**那个**设计模式。

##### 机器的崛起（MVC）

在 Web 之前，桌面电脑可以运行不需要网络通信的程序，例如电子制表软件或者字处理软件。这种应用可以在这个庞大并且强壮的桌面机器中做任何它需要做的事。

在九十年代早期，桌面电脑变得更快更好更强。与此同时，Web 苏醒了。人们开始觉得把庞大强壮的桌面应用（fat app）和与网络相连的 client / server 应用（thin app）混合在一起可以使它们各自发挥最大的能量。这种混合应用——哑终端的对立——叫做 smart 应用（smart app）。

有许多商业向的 smart 应用被创造出来，不过最早的例子来自于电脑游戏。大规模多人在线游戏（MMOs）、第一人称射击游戏和即时战略游戏都是 smart 应用，这些游戏中的信息（data model）通过一个服务器在不同的机器间传递。这时客户端比仅仅是显示信息多做了*大量的*工作。它执行大部分的处理任务（controls）并且把数据转换成要显示的东西（view）。

这种设计模式很简单，但十分有效率。它被叫做Model View Controller（MVC）模式。

![MVC](http://ww2.sinaimg.cn/large/6ad06ebbjw1e5etiuye6dj20j604gq3a.jpg)

Model 就是所有的数据。在 smart 应用的环境（context）中，model 由服务器提供。客户端发出向服务器获取数据的请求。一旦客户端获得 model，它就会在这些数据上执行动作或者逻辑，然后准备好要显示在屏幕上的东西。应用的这部分（请求服务器、修改数据 model 和准备显示的数据）称为 controller。Controller 给 view 发送命令，而 view 负责显示信息并且当某些屏幕事件（例如按钮被按下）发生时向 controller 报告。Controller 收到这个反馈后，执行逻辑部分，然后更新 model。如此循环，周而复始。

由于 web 浏览器一开始就被建造成了“哑终端”，把 web 浏览器当 smart 应用来用显然不靠谱。作为替代，smart 应用被放在诸如 Microsoft .Net、Java 或者 Macromedia（现在是Adobe）Flash 这些框架中运行。你只要装好了这些框架，就可以访问一个 web 页面来下载和运行 smart 应用。

有时你可以在浏览器中运行这个应用，有时你也可以先下载它，不管怎样，你在运行一种新型的 web 应用，这个应用可以和服务器通讯并且分担工作量。

##### 浏览器长大了（MVVM）

在千禧年初，MVC模式开始有了新变化（a new twist on the MVC pattern started to emerge）。开发者们意识到，对于相互连接的/企业级的“smart 应用”，实际上存在一个嵌套的MVC模式。

服务器（controller）使用业务对象在数据库信息（model）上处理业务逻辑，然后把这些信息交给客户端应用（一个“view”）。

客户端收到来自服务器的信息并把它当做自己的“model”。然后客户端就会和一个 controller 一样，执行逻辑并且发送信息给 view 以在屏幕上显示。

也就是说，服务器 MVC 中的“view”是第二个 MVC 中的“model”。

![view as model](http://ww2.sinaimg.cn/large/6ad06ebbjw1e5ev4v02mcj20j804naag.jpg)

想法随之而来，“为啥在两个时停下？”。一个应用没理由不能拥有*多个*嵌套的MVC，其中每个view都是下一个MVC的model。事实上，在客户端这边，其实有个很好的理由来这样做。

分离实际的显示逻辑（例如“这儿是提交按钮”、“文本框的改变值”）和客户端对象逻辑（例如“用户可以提交这个记录”、“电话号码被改变了”）可以让绝大部分的代码得到重用。对象逻辑可以被移植到不同的应用中，而你所需要做的只是针对不同的应用或设备来改写显示逻辑以扩展相同的 model 和 controller。

从 2004——2005 年，这个想法被 Martin Fowler 和 Microsoft（称为Model View View-Model）精炼和改进以供 smart 应用（称为presentation model）使用。尽管严格意义上和嵌套 MVC 不是一个东西，MVVM 设计模式把嵌套 MVC 的概念应用在了前端应用中。

![MVVM](http://ww2.sinaimg.cn/large/6ad06ebbjw1e5fge3mqxij20j704zweq.jpg)

随着浏览器技术（HTML和JavaScript）的成熟，直接在HTML页面中构建使用MVVM设计模式的smart应用成为可能。

#### 伟大的Meteor出现了！

Meteor 将 MVVM 模式提升到下一层次。通过应用 handlerbar.js（或其他模板库）模板和使用立即更新，真正让 web 应用像一个完整且鲁棒的 smart 应用。

让我们来看看 Meteor 是如何实现这些概念的，然后开始用这些来创建咱们的 Lending Library 应用。

##### 缓存的且同步的数据（model）

Meteor支持一个在客户端和数据库相同的、缓存的且同步的数据model。

![cached-and-synchronized data model](http://ww4.sinaimg.cn/large/6ad06ebbjw1e5fh0rokchj208i06kaa5.jpg)

当客户端注意到数据model的一个变化时，它首先在本地缓存这个变化，然后尝试和服务器同步。与此同时，它也监听来自服务器的变化。这使得客户端有一份数据model的副本，因此可以很快地在屏幕上显示这些变化而不用等待服务器的响应。

另外，你可能注意到了这就是MVVM设计模式刚开始时内部有一个嵌套MVC的样子。换句话说，服务器发布数据变化，将这些数据变化当做自己MVC模式中的“view”。客户端订阅这些变化，把这些变化当做自己MVC模式中的“model”。

![model as view again](http://ww2.sinaimg.cn/large/6ad06ebbjw1e5fhe2et20j20es0dbmxj.jpg)

这个代码例子在Meteor中非常简单（当然你也可以把它弄复杂以获得更多的控制权）：

``` js
var lists = new Meteor.Collection("lists");
```

这句话声明了`lists`数据model。客户端和服务器各自都有一个版本的这个model，但它们对待各自版本的方式并不同。客户端会订阅服务器声明的改变，并相应地更新自己的model。服务器会发布改变，同时监听来自客户端的改变请求，并根据这些请求更新自己的model（主副本）。

哇。短短一行代码干了这么多！当然还有更多，但这超出了本章的范围，所以让我们继续。

> 想要更好地理解Meteor数据同步，请访问Meteor文档的*[Pushish and subscribe](http://docs.meteor.com/#publishandsubscribe)*部分。

##### 模板化的HTML（view）

Meteor通过模板来渲染HTML。

HTML模板也叫做view数据绑定(view data bindings)。简单来说，view数据绑定就是当一块共享的数据改变时，它会被显示成不同的东西。

在HTML代码中有个占位符。根据变量值，不同的HTML代码会放在这个占位符处。如果变量的值发生变化，占位符中的代码也会相应改变，由此创建了新的view。

让我们看一个十分简单的数据绑定例子——其实从技术上说用不到Meteor——来解释一下刚才所说的。

在`LendLib.html`中，你会看到一个HTML（Handlebar）模板表达式：

``` html
<div id="categories-container">
    { {> categories} }
</div>
```

*（使用时应去掉大括号间的空格，下同）*

这个表达式是HTML模板的一个占位符，在下面可以找到它：

``` html
<template name="categories">
    <h2 class="title">my stuff</h2>…
```

`{ {> categories} }`基本上就是说“把模板categories里面不管啥东西都放在这儿”。相应名字的HTML模板提供了这些。

如果你想看看数据改变时显示怎样相应地变化，把`h2`标签改成`h4`，然后保存：

``` html
<template name="categories">
    <h4 class=“title”>my stuff</h4>…
```

你会在浏览器中看到效果（“mystuff”变小了）。这就是模板——或者叫view数据绑定——的效果！把`h4`改回`h2`并保存。除非你喜欢现在这样。随意了我不管……好吧，现在这样很丑、很小并且看不清。说真的，你应该在被别人看到然后嘲笑你前改回来！！

恩，现在我们知道view数据绑定是什么了，让我们来看看Meteor是如何用它们的。

在`LendLib.html`的categories模板中，你会看到更多的Handlebars模板：

``` html
<template name="categories">
    <h4 class="title">my stuff</h4>
    <div id="categories" class="btn-group">
        { {#each lists} }
        <div class="category btn btn-inverse">
            { {Category} }
        </div>
        { {/each} }
    </div>
</template>
```

第一个Handlebars表达式是成对儿的`for-each`声明的一部分。`{ { #each lists} }`告诉解释器对每个`lists`集合中的元素分别执行下面的动作（在这个例子中是创建新`div`）。`lists`是一块数据。`{ { #each lists} }`是占位符。

在`#each lists`表达式中有另一个Handlebars表达式。

``` html
{ {Category} }
```

由于它在`#each`表达式中，所以`Category`是`lists`的隐式属性。就是说`{ {Category} }`和`this.Category`一样，其中`this`是指`each`循环中的当前元素。这个占位符是在说：“把`this.Category`的值加在这儿”。

现在，看看`LendLib.js`，我们会看到模板背后的值。

``` js
Template.categories.lists = function()｛
    return lists.find(…
```

在这儿，Meteor在模板`categories`中声明了一个叫做`lists`的模板变量。这个变量是个函数。这个函数返回了之前我们定义的`lists`集合中的所有数据。这行还记得不？

``` js
var lists = new Meteor.Collection("lists");
```

这个`lists`集合由声明的`Template.categories.lists`返回。所以当`lists`集合发生变化时，相应的变量会被更新，对应模板的占位符也会被改变。

让我们做做看。打开网页`http://localhost:3000`，调出浏览器控制台然后输入下面这行：

``` js
> lists.insert({Category:"Games"});
```

这会更新`lists`的数据集合（model）。模板会看到这个变化，然后更新HTML代码/占位符。`for each`循环会为`lists`中的新内容再运行一遍。你会看到下面这个屏幕：

![lists.insert](http://ww4.sinaimg.cn/large/6ad06ebbjw1e5k74si9mrj20ev0b1q3x.jpg)

就MVVM模式来说，HTML模板代码是客户端view的一部分。数据的任何变化都会被自动反映到浏览器中。

##### Meteor的客户端代码（View-Model）

之前讨论过，`LendLib.js`包含了模板变量，把客户端的model和HTML页面（view）相连。任何发生在`LendLib.js`中的、来自view或model的改变的逻辑都是View-Mode的一部分。

View-Model负责跟踪model的变化并且把这些变化表示成view能接受的形式。它也负责监听来自view的改变。

这里的改变，并不是指按钮被按下或者文本被输入。而是，模板值的变化。一个被声明的模板就是View-Model，或者叫*view的model*。

（By changes， we don’t mean a button click or text being entered. Instead, we mean a change to a template value. A declared template is the View-Model, or the *model for the view*.）

这意味着客户端controller有它自己的model（来自服务器的数据），并且它知道如何处理这个model。View也有自己的model（模板），并且它知道怎样去显示这个model。

##### 让我们创建一些模板

现在我们会看到一个MVVM设计模式的真实例子，同时继续我们的Lending Larary项目。通过控制台添加categories很有趣，但这并不是长久之计。让我们做些事使得在页面中也可以这样。

打开`LendLib.html`并在`{ { #each lists} }`表达式之前添加一个按钮。

``` html
<div id="categories" class="btn-group">
    <div class="category btn btn-inverse" id="btnNewCat">&plus;</div>
    { {#each lists} }
```

这会给页面添加一个加号按钮。

![plus button](http://ww4.sinaimg.cn/large/6ad06ebbjw1e5k760p9vuj205i05ujrh.jpg)

现在，我们想在点击按钮时把它变成一个输入框。让我们用MVVM模式并且基于模板中变量的值来实现这个功能。

添加下面这几行代码：

``` html
<div id="categories" class="btn-group">
    { {#if new_cat} }
        { {else} }
        <div class="category btn btn-inverse" id="btnNewCat">&plus;</div>
        { {/if} }
    { {#each lists} }
</div>
```

第一行`{ { #if new_cat} }`检查`new_cat`是`true`还是`false`。如果是`false`，则`{else}`部分被触发，这意味着我们还没有指定要添加什么category，所以应该依旧显示加号按钮。
在这个例子中，由于我们还没有定义`new_cat`，所以它是`false`，因此显示不会变。现在
让我们加上添加新category时的HTML代码。

``` html
<div id="categories" class="btn-group">
    { {#if new_cat} }
        <div class="category">
            <input type="text" id="add-category" value="" />
        </div>
        { {else} }
        <div class="category btn btn-inverse" id="btnNewCat">&plus;</div>
    { {/if} }
    { {#each lists} }
```

我们添加了一个输入框，当`new_cat`为`true`时它会出现。我们应该怎样让`new_cat`等于`true`呢？

还没保存的保存一下，然后打开`LendingLib.js`。首先，我们会在lists模板声明下面声明一个`Session`变量。

``` js
Template.categories.lists = function () {
    return lists.find({}, {sort: {Category: 1}});
};
// We are declaring the 'adding_category' flag
Session.set('adding_category', false);
```

然后定义新的模板变量`new_cat`，它是一个返回`adding_category`值的函数：

``` js
// We are declaring the 'adding_category' flag
Session.set('adding_category', false);
// This returns true if adding_category has been assigned a value
//of true
Template.categories.new_cat = function () {
    return Session.equals('adding_category',true);
};
```

保存更改，你会发现啥都没变。呵呵。

事实上就应该这样，因为我们还没做任何改变`adding_category`值的事。现在开始吧。

首先，我们定义单击事件，这会改变我们`Session`中的变量值。

``` js
Template.categories.new_cat = function () {
    return Session.equals('adding_category',true);
};
Template.categories.events({
    'click #btnNewCat': function (e, t) {
        Session.set('adding_category', true);
        Meteor.flush();
        focusText(t.find("#add-category"));
    }
});
```

看这行：

``` js
Template.categories.events({
```

这行声明了categories模板中有一个事件。

下一行：

``` js
'click #btnNewCat': function (e, t) {
```

这行是说我们在找一个`id="btnNewCat"`的HTML元素（我们之前在`LendingLib.html`中创建过）上的单击事件。

``` js
Session.set('adding_category', true);
Meteor.flush();
focusText(t.find("#add-category"));
```

我们设置`Session`变量中的`adding_category=true`，我们flush这个DOM（消除不靠谱的因素），然后我们把焦点设置在`id="add-category"`的输入框上。
最后一件事，添加助手函数`focusText()`。在`if (Meteor.isClient)`函数的关闭标签前，添加下面这些代码：

``` js
/////Generic Helper Functions/////
//this function puts our cursor where it needs to be.
function focusText(i) {
    i.focus();
    i.select();
};
} //------closing bracket for if(Meteor.isClient){}
```

妙！

虽然现在这依旧没什么用，但让我们暂停一会儿来回忆下刚刚发生了什么。我们在HTML页面中创建了一个条件模板，它根据一个*变量*的值而显示输入框或者加号按钮。

这个变量属于View-Model。就是说如果我们改变这个变量的值（就像我们在单击事件中做的），view就会自动更新。我们刚刚在Meteor应用中完成了一个MVVM模式！

要完成它，让我们对`lists`集合（也是View-Model的一部分，还记得不？）做些改动，并且想办法在完成时把`input`输入框隐藏起来。
首先，我们需要给`keyup`事件添加一个监听器。或者是，我们想监听到用户在输入框中输入了一些东西后按下了*回车*。这时，我们想把用户填写的category添加进去。首先，声明事件处理函数。在`#btnNewCat`的`click`事件后面添加另一个事件处理函数：

``` js
        focusText(t.find("#add-category"));
    },
    'keyup #add-category': function (e,t){
         if (e.which === 13)
         {
            var catVal = String(e.target.value || "");
            if (catVal)
            {
                lists.insert({Category:catVal});
                Session.set('adding_category', false);
            }
        }
    }
});
```

我们在单击函数后面添加了一个“，”，然后是`keyup`事件处理。

``` js
if (e.which === 13)
```

这行检查我们按下的是不是*回车*键。

``` js
var catVal = String(e.target.value || "");
if (catVal)
```

这检查输入框中是否有值。

``` js
lists.insert({Category: catVal});
```

如果有，我们想把它添加到`lists`集合中。

``` js
Session.set('adding_category', false);
```

然后我们想隐藏这个输入框，只要简单地改下`adding_category`的值就好。
还有一个东西要加，然后就完成了。如果我们点击`input`输入框之外，就应该隐藏它，并且把添加按钮放回来。我们已经知道了在MVVM模式中怎样实现它，就让我们来加个改变`adding_category`值的函数。在`keyup`事件处理后加一个逗号，插入下面这个事件处理函数：

``` js
                Session.set('adding_category', false);
            }
        }
    },
    'focusout #add-category': function(e,t){
        Session.set('adding_category',false);
    }
});
```

保存更改，让我们来试试看！在浏览器`http://localhost:3000`中，点击加号——输入单词**Clothes**然后*回车*。

你的屏幕应该像下面这样：

![input box](http://ww4.sinaimg.cn/large/6ad06ebbjw1e5k76yzj6hj208c076mxi.jpg)

如果你喜欢请随便添加categories。当然，试试点击加号按钮，输入一些东西，然后点击输入框之外也是可以的。

#### 小结

在这章中你了解了web应用的历史，并且知道了我们从传统的client/server模型演变成了全面成熟的MVVM设计模式。你看到了Meteor是如何利用模板和同步数据让事情变得易于掌握，并且给我们的view、逻辑和数据提供了清晰的分离。最后，你给Lending Library加了一些东西，做了一个添加category的按钮。这些都是通过使用View-Model的改变完成的，而不是直接编辑HTML。在下一章中，我们会真正地做一些工作，添加各种类型的模板和逻辑，实现我们的Lending Library！

