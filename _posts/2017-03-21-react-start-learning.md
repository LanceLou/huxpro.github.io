---
layout:     post
title:      "React初探"
subtitle:   "虚拟DOM 单向数据流 JSX"
date:       2017-03-20 00:00
author:     "LanceLou"
header-img: "img/post-react-start-learning.png"
catalog: true
tags:
    - JavaScript
    - ReactJS初体验
---

> A JAVASCRIPT LIBRARY FOR BUILDING USER INTERFACES

从2013年推出至今4载，React能走到今天，并且持续的发光发热，我想自有其优美，非凡之处。作者也有幸能在最近一段时间里接触了React，确实感觉React在前端业务处理，性能优化方面的非凡之处，瞎逼逼不说，写下作者的一些感想，也算是React入门的一个小总结。


## 虚拟DOM(JavaScript 对象树)

我想这是接触React所绕不过的一个地方，作者在没接触React之前，都经常在社区里面听到各种有关React的Virtual Dom的文章，说道Dom，其实是前端特别熟悉的一个名词，咱们说JavaScript起家就是做表单验证的，Dom的操作也自然是JavaScript的看家行当了。

* 传统的JavaScript Dom操作:

![traditionalDomUpdate](https://www.tuchuang001.com/images/2017/03/21/traditionalDomUpdate.png)


* 但是，在React中，咱们写的JavaScript不在是和Dom打交道，而是和Virtual Dom打交道：

![ReactDomUpdate](https://www.tuchuang001.com/images/2017/03/21/ReactDomUpdate.png)

其实说React是一个库(Lib),我觉得他更像是一个框架，亦或者说是一个系统，为什么这么说呢，因为他从"底层"给我们封装了一个层，在这层之上，我们操作Virtual Dom，这层之下，React帮我们去实现真实的Dom更新，这样做有什么好处呢？在目前这个阶段，我个人的理解是:

* 实现在全局概念下的局部更新，按需跟新。
* 封装底层具体实现，也即不必关心你下面跑的具体是那个浏览器，抑或是何种平台。

## JSX

React组件化的基础，在JavaScript构建HTML结构化页面。有效的表达React中组件间的嵌套关系。

简单的说，我觉得，JSX就是JavaScript中HTML的一种表达形式，一种使用JavaScript描述的HTML，且最终也会转化为JavaScript。那么为什么要把HTML放到JavaScript中呢。我觉得有如下几点考虑:

* 组件化，以JavaScript对象的方式(JSON)来表述标签，可重用，可嵌套。
* 嵌套，以递归渲染的方式构建出整个DOM元素树，同时，在内存中保存完整的虚拟DOM，便于实现高效DOM操作。

同时，JSX是以xml的语法来描述组件以及嵌套的，这对于前端开发人员是非常有好的。

## 单向数据流

在React中，数据的流向是单向的。

其实，我觉得，这设计到一种软件开发的工程问题，说为什么是单向数据流，单向数据流和非单向数据流的关系，我觉得这种问题就像讨论面向对象和面向过程的关系，以及基于面向对象之上的AOP编程。你说我不用单向数据流方式开发的软件就不能跑了吗？显然答案是否定的。

但凡设计到这种工程问题，脱离某一特定语言的具体实现的问题，都有一个高大上的名字来匹配，设计模式(Design Pattern)。这不是本文的主体，作者也就不再这瞎逼逼了。

其实对于JavaScript这种脚本语言啊，你说写个100行左右的小功能，结构清晰，流程合理那是末事儿的。但偏偏在这个WEB2.0时代，富应用，前端页面，或者说web app的体积蹭蹭蹭往上涨。甚至在当年Gmail的引领下，使用ajax页面无刷新，强大而复杂的SPA，你说你写个几十行百来行够吗？？？

自然，代码量的增多会带来一系列的问题，可读性方面，测试方面，数据的把控等。也自然有一系列的解决方案。如模块化开发以及随之而来的打包构建。

而数据流，也是Application里面的一个关键的环节，清晰，明了，简介的数据流有利于我们应用的结构化，同时也可以精切的定位程序的bug，数据的监控以及加工方面也方便许多。--> 数据可控性。

说道React的单向数据流，大多会提到Angular的双向数据绑定，Angular的技术细节作者没有具体去了解，但之前有照着别人的教程敲过一个Angular的聊天应用，数据层和视图层的双向绑定

>所谓的双向绑定，无非是从界面的操作能实时反映到数据，数据的变更能实时展现到界面。

```
//UI
<div ng-controller="CounterCtrl">
    <span ng-bind="counter"></span>
    <button ng-click="counter=counter+1">increase</button>
</div>

//Data
function CounterCtrl($scope) {
    $scope.counter = 1;
}
```
其实存在即合理，这里作者没有资格去谈双向绑定的缺点与不足，但可谈下React的单向数据流不是？

在React中，组件的通信的数据流是单向的，只能从顶层的组件通过props属性向下层组件传递数据，而下层组件不能向上层组件传递数据，兄弟组件也不能。单一的数据流向，确定的数据流通环节 -> 数据可控性。同时，通过State的来触发组件的重绘，props来传递属性，一气呵成。


## 结语

说实话，作为一个官方自称为View层JAVASCRIPT LIBRARY的React，我觉得React的功能还是挺强大的，同时，其在前端开发方面的独特见解与切入点也让我意识到了传统开发方式的弊端，持续学习，继续深入！！！全文为LanceLou个人观点，如有不足，还请不吝赐教！

The end

———— 2017年3月 LanceLou