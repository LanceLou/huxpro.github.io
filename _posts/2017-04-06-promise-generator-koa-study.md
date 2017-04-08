---
layout:     post
title:      "谈谈promise，generator，学习koa"
subtitle:   "promise generator koa nodejs study"
date:       2017-04-05 00:00
author:     "LanceLou"
header-img: "img/post-promise-generator-koa.jpg"
catalog: true
tags:
    - NodeJS初体验
    - JavaScript
---

> 不知道会去往何处，也不知道该去哪儿，但总归，还得走下去。
> 
> —— 成长

这几天博主在鼓弄个NodeJS爬虫的小任务，然而居然被推荐使用koa框架，话说Express都还没玩熟悉呢。不过存在即合理，既然有人推荐使用koa，自然有其优秀之处，怎么说？折腾折腾呗。

## 开始

正所谓翻山越岭找遍各大论坛，博客，寻找各种course， demo， example。 自从有了NodeJS的前车之鉴，作为入门，谁再推荐我去RTFM， 跟谁干瞪眼。 说人话就是：我觉得学一门技术，还是不要一开始就看文档，我比较喜欢的是，先动手，然后再带着问题，带着自己的理解去读文档，你会发现，其乐无穷，原来如此，略懂略懂(此处省略瞎逼逼)......

OK，废话不多说，首先需要回答几个问题，Koa是啥，Koa和Express有什么区别呢。

#### what's

> 由 Express 原班人马打造的 koa，致力于成为一个更小、更健壮、更富有表现力的 Web 框架。使用 koa 编写 web 应用，通过 ***组合不同的 generator，可以免除重复繁琐的回调函数嵌套，并极大地提升常用错误处理效率***。Koa 不在内核方法中绑定任何中间件，它仅仅提供了一个轻量优雅的函数库，使得编写 Web 应用和API变得得心应手。

当然这是V2的时候所介绍的，在即将发布的V3里面将去除对generator函数的支持，所以现在在跑KOA2.*的时候会报deprecated的警告。

> koa deprecated Support for generators will be removed in v3. See the documentation for examples of how to convert old middleware https://github.com/koajs/koa/blob/master/docs/migration.md

#### why

说道Koa和Express的区别，我想各位听到的最多的声音就是，koa以一种同步的方式去编写异步代码，省去回调函数，转而代之的是generator(v1/v2), promise， 以及 asynchronous functions（ES2016），同时，可以通过try，catch语法来捕获错误。而在Express中的画风就是各种回调咯(其实作为前端汪，我觉得回调并不是那么难接受，当然回调地域除外。只是在Koa中，他们的目标是消除回调，注意是消除，不是减少)。同时，在Koa中，中间件的传递循序如下图。


![koa-middleware.jpeg](/img/in-post/koa-middleware.jpeg)


***当然，讨论Koa以及Express不是本文的主体，博主更感兴趣的是Koa中对Generator函数和promise函数的使用***

## Promise

在前几年的时候，Prmise作为ES2016的新标准，还是个很火的东西，社区里各种讨论Promise in ES6，虽然社区里早有了类似的异步解决方案。

Promise，就如其名，一种允诺，里面保存了行为，一般是未来才会完成的动作(异步)。返回Promise对象，通过在返回的对象上面调用then方法来进行对应的异步的结果的处理。算是对异步方法的一种封装。如promise包装的ajax处理函数:

```JavaScript
var getJSON = function(url) {
  var promise = new Promise(function(resolve, reject){
    var client = new XMLHttpRequest();
    client.open("GET", url);
    client.onreadystatechange = handler;
    client.responseType = "json";
    client.setRequestHeader("Accept", "application/json");
    client.send();

    function handler() {
      if (this.readyState !== 4) {
        return;
      }
      if (this.status === 200) {
        resolve(this.response);
      } else {
        reject(new Error(this.statusText));
      }
    };
  });

  return promise;
};

getJSON("/posts.json").then(function(json) {
  console.log('Contents: ' + json);
}, function(error) {
  console.error('出错了', error);
});
```

同时，promise也可以部署统一的error handler方法进行异常捕获。当然，如果Promise的功能只在上面提到的几个地方，也就不会有promise了。其实但凡JavaScript的异步解决方案，你都会发现有一个叫异步流程控制的解决方案。也就是说流水线上的异步如何处理，并发的异步如何处理？

#### 异步流程-回调嵌套

很容易理解，在异步方法的回调函数里面发起另一个异步操作，且很基础，因为百变不离其宗。缺点也很明显，流程多了，嵌套就多了。以简单的http server处理为例。

```JavaScript

function parseBody(req, callback) {
  //根据http协议从req中解析body
  callback(null, body);
}
function checkIdInDatabase(body, callback) {
  //根据body.id在Database中检测，返回结果
  callback(null, dbResult);
}
function returnResult(dbResult, res) {
  if (dbResult && dbResult.length > 0) {
    res.end('true');
  } else {
    res.end('false')
  }
}
function requestHandler(req, res) {
  parseBody(req, function(err, body) {
    checkIdInDatabase(body, function(err, dbResult) {
      returnResult(dbResult, res);
    });
  });
}
```

#### 异步流程-Promise

通过Promise，你就不必通过函数嵌套了，直接的then方法:

```JavaScript
somFuncPromise_1(args).then(function(data) {
  return somFuncPromise_2(data);
}).then(function(result) {
  //result hadnler
}, function(err){
  console.log("Rejected: ", err);
});
```

前提是Promise依旧返回一个Promise，从而实现链式的then调用。somFuncPromise_1方法，somFuncPromise_2方法都是立即返回一个Promise。即将回调包装成Promise。

将回调改成Promise还是挺简单的:

```JavaScript
let somFuncPromise = (someArgs) => (
  new Promise((resolve, reject) => {
   	 //some operate...
    somFuncAsync((error, data) => {
      if (error) reject(error);
      else resolve(data);
    });
  })
)
```

此处需要注意: then方法回调函数的调用代表着Promise状态的改变，不管是出错还是完成异步。

## Generator

又是一个ES6新增的属性，类似于一个状态机，内部保存有状态，通过暴露的方法进行状态机里外的参数传递，来实现状态机内部与状态机外部的数据交互。

```JavaScript
function* f() {
  for(var i = 0; true; i++) {
    var reset = yield i;
    if(reset) { i = -1; }
  }
}

var g = f();

g.next() // { value: 0, done: false }
g.next() // { value: 1, done: false }
g.next(true) // { value: 0, done: false }
```

yield后面的表达式的结果将会返回给外部的next()函数调用，而next函数的的参数将会作为状态机内部的上一个yield表达式的左值。由此就产生了状态机内外的数据交互。

## Generator异步"同步化"

在Generator中，我们需要了解的一个知识点就是执行权的问题，当我们的Generator函数执行到yield后面的语句时，如果yield后面是一个函数，执行权就会从Generator函数转移到对应的函数，当我们调用Generator的'实例'的next方法时，执行权又从对应的函数转回Generator函数。

理解上面这个细节，对于解释Genrator在异步方面的改进就容易多了。无论是Thunk函数还是Promise版本。

#### Thunk函数与Generator

JavaScript中的Thunk函数类似于偏函数(柯里化)，最终返回经过Thunk加工版本的异步函数只接收一个回调函数作为参数。同时，我们也可以写一个Thunk函数包装器:

```
var Thunk = function(fn){
  return function (){
    var args = Array.prototype.slice.call(arguments);
    return function (callback){
      args.push(callback);
      return fn.apply(this, args);
    }
  };
};
```

通过Thunk函数的包装，我们就可以简单的实现异步函数加Generator函数的流程管理，

```
var fs = require('fs');
var thunkify = require('thunkify');
var readFileThunk = thunkify(fs.readFile);


var gen = function* (){  //同步化的代码，似乎看不出异步的味道咯
  var r1 = yield readFileThunk('/etc/fstab');
  console.log(r1.toString());
  var r2 = yield readFileThunk('/etc/shells');
  console.log(r2.toString());
};

//流程管理
var g = gen();

var r1 = g.next();
r1.value(function (err, data) {
  if (err) throw err;
  var r2 = g.next(data);
  r2.value(function (err, data) {
    if (err) throw err;
    g.next(data);
  });
});
```

通过最终对fs.readFile的thunkify，当readFileThunk('someFiles')时，我们得到的是一个还需要一个回调函数的result(即r1.value)，通过传入回调函数，在回调函数里面调用g.next(data)(关键)，从而实现了控制权的交回以及参数的传递。

当然，上面是手动的流程管理，流程管理的自动化，才是本节的主体:

```
//接收参数Generator函数
function run(fn) {
  var gen = fn();

  function next(err, data) {
    var result = gen.next(data); //自动交回执行权，参数传递
    if (result.done) return; //状态机执行结束，结束自动流程
    result.value(next);  //传入回调
  }

  next();
}
```

如上，通过我们的next函数，来实现对应的流程管理。

#### Promise与Generator

那么既然Thunk函数通过callback与Generator实现异步控制，自然Promise也可以，只不过一个是在callback里面，而一个是在Promise的then方法中。

类似就可以推出基于Promise的自动流程管理

```JavaScript
function run(gen){
  var g = gen();

  function next(data){
    var result = g.next(data); // 当Genrator未完时，接收到一个Promise(result.value)
    if (result.done) return result.value;
    result.value.then(function(data){
      next(data);
    });
  }

  next();
}

run(gen);
```

***其实不管是基于Promise还是Thunk，都是在异步操作完成的时候将执行权交回Generator***



## How about Koa

Koa的中间件就是以Generator的方式编写的，并且保存了一个gen的栈，yeild后面的Thunk也会转成Promise。貌似Koa将在最新版里面移除Koa，官方也不再推荐使用Genrator。但这并不阻碍咱们对其的理解是不。

Koa方面作者还是个初学者，还在慢慢积累，这方面就不做过多的班门弄斧了。


***以上就是笔者自己的一些理解与总结，相关代码有参考阮老师的ES6教程，并添加了自己的一些理解***

## Refers

* [ES6 Generator函数](http://es6.ruanyifeng.com/#docs/generator-async)
* [Thunkify源码，Co源码以及与Koa的理解](http://www.jianshu.com/p/e9640b41c4cd)


—— Tth End

—— LanceLou 2016.04.07