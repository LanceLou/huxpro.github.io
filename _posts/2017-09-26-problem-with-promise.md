---
layout:     post
title:      "【译】一些关于promise的问题"
subtitle:   "JavaScript，Promise"
date:       2017-09-26 00:00
author:     "LanceLou"
header-img: "img/post-problem-with-promise.png"
catalog: true
tags:
    - JavaScript
---

原文链接: [https://pouchdb.com/2015/05/18/we-have-a-problem-with-promises.html](https://pouchdb.com/2015/05/18/we-have-a-problem-with-promises.html)


各位JavaScript的开发者们，是时候，我们必须承认，对于Promise，we have a problem。

当然，这和promise本身没有什么关系，在 [A+ spec](https://promisesaplus.com/)规范中所定义的promise，是非常了不起的。

在过去的几年中，看到无数的开发者对PouchDB和其他类似于PouchDB这样的promise强依赖的库所表现出的困惑的后，我意识到，在此所存在的最大的问题是：

大多数使用promise的开发者并没有真正了解它。

如果你觉得这难以置信，你可以看下我最近[推的这个问题(I recently posted to Twitter)](https://twitter.com/nolanlawson/status/578948854411878400): 

## Q: 下面四个promise之间的区别是啥？

```JavaScript
doSomething().then(function () {
  return doSomethingElse();
});

doSomething().then(function () {
  doSomethingElse();
});

doSomething().then(doSomethingElse());

doSomething().then(doSomethingElse);
```

如果你知道这个问题的结果，那么祝贺你: 你是一名promise方面的专家，自然，本文下面的部分你也无需再读了。

而对于其他99.99%的人。在我的推特下面的回复，没有一个可以解决这个问题。

问题的答案我将在文末给出，但在这之前，我希望解释下为什么promise是如何的微妙而不可捉摸，并且为毛我们中间的大多数，被其所坑。同样我也会提供一些我所认为的一些不错的视角，或者你可以认为是一些雕虫小技。来使得promise变得更加易于理解。阅完这篇文章之后，我想它就不会如此困难了。

在此之前，我们需要指出，有关于promise的一些假设，或者你所认为的，其实并非如此，值得怀疑。

## 为什么是promise

如果你有读到promise相关的文章，你会发现在这些文章中必会提及[回调地狱相关的知识点](https://medium.com/@wavded/managing-node-js-callback-hell-1fe03ba8baf), 随着回调函数一步步的往下层嵌套，我们的代码也主键延伸到了屏幕的最右边。

promise确实解决了我们上面提到的问题，但其实不只是我们代码风格，抑或布局的改变。回调函数带给我们的最大的问题在于，它剥夺了我们使用抑或如我们愿使用return和throw语句的权利。取而代之的是，我们整个程序的流程将会建立在一个副作用之上: 即一个函数可以随意的去掉用另一个函数。

实际上，回调函数的使用在某种情况下可能会更加的糟糕：那就是它们的调用栈。编写没有清晰的调用栈的程序就像开车不系安全带。在你真正碰到调用栈的问题之前，你永远不会知道，没有清晰的函数调用栈的程序，是多么的糟糕。

promise所真正给我们的，是在这之前我们写异步代码时所失去的return，和throw语句，以及调用栈。但你需要如何合理的使用之，从而合理的利用之。

## 初学者的错误(Rookie mistakes)

有人尝试[通过卡通来解释promise](http://andyshora.com/promises-angularjs-explained-as-cartoon.html), 或者从字面上来解释(noun-oriented): 他是一个异步操作的结果，并且你可以传递它。

我并没有找到对我非常有帮助的解释，于我来说，promise与代码结构和程序流程有关。所以我认为说清楚promise的最好的方式就是，给出一些大家常犯的错误，同时解释如何去fix它们。我把这些错误称为"初学者的错误(Rookie mistakes)"。

小小的离一下题: "promise"对于不同的人可能有不同的意思，但对于这篇文章，我将只探讨其[官方释义, Promises/A+](https://promisesaplus.com/)。也即许多现代浏览器所实现的window.Promise。尽管并不是所有的浏览器都支持Promise，多亏了polyfill，你可以通过导入polyfill来实现在不支持的浏览器中的兼容性。

#### Rookie mistake #1: the promisey pyramid of doom

看看许多PouchDB的用户如何使用这个基于Promise的库，你会发现许多的使用Promise的bad pattern。其中出现的最多的就是下面的代码:

```JavaScript
remotedb.allDocs({
  include_docs: true,
  attachments: true
}).then(function (result) {
  var docs = result.rows;
  docs.forEach(function(element) {
    localdb.put(element.doc).then(function(response) {
      alert("Pulled doc with id " + element.doc._id + " and added to local db.");
    }).catch(function (err) {
      if (err.name == 'conflict') {
        localdb.get(element.doc._id).then(function (resp) {
          localdb.remove(resp._id, resp._rev).then(function (resp) {
// et cetera...
```

ok，最后这就变成了你可以把promise当成回调函数来使用，杀鸡焉用牛刀，但你确实可以这么做，不是嘛。

你可别认为这只是新手才会犯的错误，如果我告诉你，上面的代码来自于一篇黑莓公司的开发者的博客(the official BlackBerry developer blog)呢。已经养成的写回调函数的习惯是很难去除的。

更好的风格:

```JavaScript
remotedb.allDocs(...).then(function (resultOfAllDocs) {
  return localdb.put(...);
}).then(function (resultOfPut) {
  return localdb.get(...);
}).then(function (resultOfGet) {
  return localdb.put(...);
}).catch(function (err) {
  console.log(err);
});
```

这种方式叫做链式调用(composing promise)，也是promise的强大的一个地方。

#### Rookie mistake #2: WTF, how do I use forEach() with promises?

这也是许多人对promise的理解开始出错的地方，当它们写到它们所熟悉的forEach()循环(或者说for循环，抑或while)，它们就不知道如何将循环和promise结合。从而写出了如下这样的代码:

```
// I want to remove() all docs
db.allDocs({include_docs: true}).then(function (result) {
  result.rows.forEach(function (row) {
    db.remove(row.doc);  
  });
}).then(function () {
  // I naively believe all docs have been removed() now!
});
```

代码的问题在于: 传入第一个then的方法实际上返回的是undefined，这就意味着传入第二个then的方法并不会等到在所有的documents上调用了db.remove函数完成之后才执行。实际上，它根本就不会等。

解决这个问题的关键在于，forEach()/for/while并不是你所需要的逻辑。你应该使用Promise.all():

```JavaScript
db.allDocs({include_docs: true}).then(function (result) {
  return Promise.all(result.rows.map(function (row) {
    return db.remove(row.doc);
  }));
}).then(function (arrayOfResults) {
  // All docs have really been removed() now!
});
```

#### Rookie mistake #3: forgetting to add .catch()

这是另一个经常会出现的问题，过于乐观的相信promise里面永远不会抛出异常。许多的开发者经常忘记添加加catch。不幸的是，这意味着在promise中抛出的错误都会被淹没(swallowed)，外界无法感知。你也不会在控制台看到他们。这对于debug来说是非常痛苦的。

为例避免这种痛苦的情景，我已经养成了习惯性的给我的promise链加如下代码的习惯:

```
somePromise().then(function () {
  return anotherPromise();
}).then(function () {
  return yetAnotherPromise();
}).catch(console.log.bind(console)); // <-- this is badass
```

即使你永远也不希望会出现error，习惯性的添加catch()是明智的。他能让你的生活变得更轻松。

#### Rookie mistake #4: using "deferred"

这是我见得最多的一个错误，我都不想在提起它。

总之，promise的发展有很长的历史，且故事满满。其也让JavaScript社区花了很长的时间来接纳它，在之前，jQuery和Angular也都是使用deferred 模式，现在也被更换为使用ES6的promise规范，同时这个规范也被许多的库实现，像RSVP, Bluebird, Lie, and others.

所以如果你在代码里使用了deferred，那么我想说你做了一些糟糕的事情。在这里我将告诉你如何解决它:

首先，许多promise的库提供给我们方法来将第三方的promise引入，例如，$q模块提供了$q.when()方法来包装非$q（non-$q）的promise。所以使用angular的开发者可以使用这种方法来包装PouchDB的promise。

```
$q.when(db.put(doc)).then(/* ... */); // <-- this is all the code you need
```

另外一种方法是使用[揭示构造器模式(revealing constructor pattern)](https://blog.domenic.me/the-revealing-constructor-pattern/),在包装非promise标准的API上非常实用。例如：我们用这种方式来包装我们nodejs的fs.readFile():

```JavaScript
new Promise(function (resolve, reject) {
  fs.readFile('myfile.txt', function (err, file) {
    if (err) {
      return reject(err);
    }
    resolve(file);
  });
}).then(/* ... */)
```

> 了解更多关于为什么使用deferred是一钟反模式，可以访问[the Bluebird wiki page on promise anti-patterns](https://github.com/petkaantonov/bluebird/wiki/Promise-anti-patterns#the-deferred-anti-pattern)

#### Rookie mistake #5: using side effects instead of returning

我们来看下下面的代码(有问题吗？):

```JavaScript
somePromise().then(function () {
  someOtherPromise();
}).then(function () {
  // Gee, I hope someOtherPromise() has resolved!
  // Spoiler alert: it hasn't.
});
```

ok，一个很好的出发点，这让我们可以讨论你所需要了解的关于promise的许多事情。

敲黑板，我下面所要说的是非常关键的一点，一旦你理解了这个点，将会有效的阻止我所提到的错误的发生，你有准备好吗？

如我之前提到的，promise的魔力在于，它让我们可以重新使用return和throw语句，那么，在实际使用的过程中，它的表现又是什么样的呢。

每一个promise对象提供给我们一个then()方法(or catch(), 当然这不过是then方法的语法糖)，现在我们处在一个then方法里面

```JavaScript
somePromise().then(function () {
  // I'm inside a then() function!
});
```

我能能够在这个地方干啥:

* 返回另一个promise
* 返回一个同步的值(或者undefined)
* 抛出一个同步的异常

就这些，一旦你理解的这个点，你就会理解promise。我们现在来逐个分析。

**1. 返回另一个promise**

这是一个常见的promise的用法，就如我们前面级联版的promise:

```JavaScript
getUserByName('nolan').then(function (user) {
  return getUserAccountById(user.id);
}).then(function (userAccount) {
  // I got a user account!
});
```

需要强调的是，我们将第二个promise"返回了"，这个return是关键的，如果我们不使用return，那么getUserAccountById将会是一个副作用(side effect)，并且下一个方法将会接收到undefined，而不是我们所希望的userAccount。

**2. 返回一个同步的值**

返回一个undefined通常是一个错误的做法，但是返回一个同步的值却是一个不错的方法来将同步的代码转化为promise的代码。我们我们有在内存中的users的cache，我们就可以这样做:

```JavaScript
getUserByName('nolan').then(function (user) {
  if (inMemoryCache[user.id]) {
    return inMemoryCache[user.id];    // returning a synchronous value!
  }
  return getUserAccountById(user.id); // returning a promise!
}).then(function (userAccount) {
  // I got a user account!
});
```

是不是感觉很良好，第二个方法无需去关注userAccount到底是异步还是同步获取的，而且第一个方法也可以随意的去选择返回一个同步的or一个异步的值。

但不幸的是，在JavaScript语言中，有一个不方便的一面，没有返回操作的函数调用(non-returning)将会默认的返回undefined，这就意味着JavaScript引擎无法区分我们是需要返回一个值(却返回的undefined)，和我们本身就是不打算返回。从而会出现这种情况，当我们本来打算返回某个值的，却意外的产生了side effect。

就因为这个原因，我养成了在then方法中要么返回要么抛错的习惯。我也建议你像我这么做。

**3. 抛出一个同步的错误**

说到throw，我想throw也是使得promise变得如此美好的一面。它可以让我们很方便的去做某些事情，比如当用户登出的时候抛出一个同步的异常。

```JavaScript
getUserByName('nolan').then(function (user) {
  if (user.isLoggedOut()) {
    throw new Error('user logged out!'); // throwing a synchronous error!
  }
  if (inMemoryCache[user.id]) {
    return inMemoryCache[user.id];       // returning a synchronous value!
  }
  return getUserAccountById(user.id);    // returning a promise!
}).then(function (userAccount) {
  // I got a user account!
}).catch(function (err) {
  // Boo, I got an error!
});
```

当用户已经登出的时候，我们的catch()将会接收到一个同步的异常，同样，如果链上任何一个promise被reject了，它也会收到一个异步的异常。再次强调，catch()并不关心异常是异步的还是同步的。

这些特性是非常有用的，因为它可以帮助我们在开发的过程中定位错误。例如： 如果我们在then方法的许多地方都有使用JSON.parse()这个方法，如果JSON是无效的，它将有可能会抛出一个同步的异。而在回调函数的方式里面，异常会被沉没(swallowed)，在promise中，通过catch(),我们可以很方便的处理。

## 高级错误(Advanced mistakes)

Okay，现在你已经学到了一些让我们写promise更加容易的方法，那么现在，我们来讨论一些特殊情况。

我把这些错误定义为高级的错误，因为我只在非常熟练promise的开发者所写的代码中才有看到这些。但在此我们依旧需要探讨它，因为只有这样，我们才能解决我在文首所提到的问题。

#### Advanced mistake #1: not knowing about Promise.resolve()

就像我在前面提到的，promise非常擅长将同步代码包装成异步代码，但是，如果你发现你写了很多这样的代码:

```JavaScript
new Promise(function (resolve, reject) {
  resolve(someSynchronousValue);
}).then(/* ... */);
```

或许你可以尝尝更加简洁的办法:

```JavaScript
Promise.resolve(someSynchronousValue).then(/* ... */);
```

同样，在捕获同步异常方面，promise也是非常有用。我已经养成了几乎将我的所有的返回promise的API包装成如下的样子:

```JavaScript
function somePromiseAPI() {
  return Promise.resolve().then(function () {
    doSomethingThatMayThrow();
    return 'foo';
  }).then(/* ... */);
}
```

请记住，任何可能会抛出同步异常的代码，都极有可能会变成几乎无法debug的沉没的异常。但是如果你把这些东西用Promise.resolve()包装下，这些异常就一定能够被捕获。

同样，你也可以用Promise.reject()来返回一个立即reject的异常。

#### Advanced mistake #2: catch() isn't exactly like then(null, ...)

在此之前，我有说catch方法就是个语法糖，所以下面这两个代码片段是相等的:

```JavaScript
somePromise().catch(function (err) {
  // handle error
});

somePromise().then(null, function (err) {
  // handle error
});
```

但是，这并不意味着下面的两个代码片段是相等的:

```JavaScript
somePromise().then(function () {
  return someOtherPromise();
}).catch(function (err) {
  // handle error
});

somePromise().then(function () {
  return someOtherPromise();
}, function (err) {
  // handle error
});
```

如果你怀疑我所说的，那么你可以想下如果第一个函数抛出了一个异常，会发送什么:

```
somePromise().then(function () {
  throw new Error('oh noes');
}).catch(function (err) {
  // I caught your error! :)
});

somePromise().then(function () {
  throw new Error('oh noes');
}, function (err) {
  // I didn't catch your error! :(
});
```

当你使用then(resolveHandler, rejectHandler)这种格式时，rejectHandler函数并不能捕获到resolveHandler所抛出的异常。

就是因为这个原因，所以我从来不使用then()方法的第二个参数，并且一直使用catch()函数。

#### Advanced mistake #3: promises vs promise factories

如果你希望循序的执行一系列的promise。但并不是像promise.all()那样并行的执行这些promise。

你有可能会写出这样的代码:

```JavaScript
function executeSequentially(promises) {
  var result = Promise.resolve();
  promises.forEach(function (promise) {
    result = result.then(promise);
  });
  return result;
}
```

然而，这些promise并不会以你所希望的方式执行，你传入executeSequentially的这些promise依旧是在并行的执行。

原因在于你其实并不是希望去运行完整个promise的数组。依据promise的规范，只要promise被创建，它就会开始执行。所以我想，你真正需要的是一个promise工厂的数组(array of promise factories):

```JavaScript
function executeSequentially(promiseFactories) {
  var result = Promise.resolve();
  promiseFactories.forEach(function (promiseFactory) {
    result = result.then(promiseFactory);
  });
  return result;
}
```

当然，promise 工厂函数其实是非常简单的，它仅仅就是一个返回promise的函数:

```JavaScript
function myPromiseFactory() {
  return somethingThatCreatesAPromise();
}
```

是不是很好奇这是如何实现的？promise 工厂只有当你调用它的时候才会去创建一个promise，同时关键的是，当其和then方法进行配合的时候，神奇的事情就发生了(此处读者可自行体会下)。

(ps: 作者原文译: 看到上面的executeSequentially函数，同时将myPromiseFactory作为then方法的第一个参数，然后，我想，在你的脑瓜子里面，是不是有一个小灯泡被点亮啦。那么此时，我想你已经悟道了promise的一些东西blabla)。

#### Advanced mistake #4: okay, what if I want the result of two promises?

通常，一个promise依赖于另一个promise的输出，但是有时我们却同时需要两个promise的输出，例如：

```JavaScript
getUserByName('nolan').then(function (user) {
  return getUserAccountById(user.id);
}).then(function (userAccount) {
  // dangit, I need the "user" object too!
});
```

作为一名有追求的JavaScript开发者，自然是嫌弃回调地狱的，那么我么可能会在更高一级的作用域中去存储我们的user对象。

```JavaScript
var user;
getUserByName('nolan').then(function (result) {
  user = result;
  return getUserAccountById(user.id);
}).then(function (userAccount) {
  // okay, I have both the "user" and the "userAccount"
});
```

This works，毫无疑问，但我个人认为上面的方式有点凑合。就我个人而言，请抛下你对回调地狱的偏见，或许，在某种场景下，它其实没那么差：

```JavaScript
getUserByName('nolan').then(function (user) {
  return getUserAccountById(user.id).then(function (userAccount) {
    // okay, I have both the "user" and the "userAccount"
  });
});
```

至少，就目前来看，还是ok的，如果你觉得缩进是个问题，那么或许你可以想JavaScript开发者几十年前就已经在做的那样，将匿名函数抽成命名函数:

```JavaScript
function onGetUserAndUserAccount(user, userAccount) {
  return doSomething(user, userAccount);
}

function onGetUser(user) {
  return getUserAccountById(user.id).then(function (userAccount) {
    return onGetUserAndUserAccount(user, userAccount);
  });
}

getUserByName('nolan')
  .then(onGetUser)
  .then(function () {
  // at this point, doSomething() is done, and we are back to indentation 0
});
```

随着你的promise代码逐渐变得复杂，你或许会发现越来越多的命名函数被你抽取了出来。但我发现其实这种代码还是不乏美感的(aesthetically-pleasing);

#### Advanced mistake #5: promises fall through

最后，这个错误自然是和文首所提到的promise的迷惑(promise puzzle)相关的。这是一个非常难懂的用例，并且它可能永远也不会在你的代码里面出现，但它着实震惊到了我。

你认为下面的代码会打印出啥？

```JavaScript
Promise.resolve('foo').then(Promise.resolve('bar')).then(function (result) {
  console.log(result);
});
```

如果你认为它会打印出bar，那么你就错了，它实际上打印的是foo！

为什么会这样呢？其实，当你传给then方法一个非函数的参数时(如promise)，它实际上把这钟情景当成then(null)，这导致前一个promise的结果往下传递，你可以如下这项来验证我所说的:

```JavaScript
Promise.resolve('foo').then(null).then(function (result) {
  console.log(result);
});
```
当然，不管你中间插入多少的then(null)，最后依旧会打印出foo。

这实际上就回到了我前面所提到了关于promises vs promise factories中的一个点。总之，你可以直接把promise传递给then()方法，但它并不会如你所希望的那样去运行。then()所支持的是传递一个函数，所以你最可能想去做的是:

```JavaScript
Promise.resolve('foo').then(function () {
  return Promise.resolve('bar');
}).then(function (result) {
  console.log(result);
});
```

这将会打印bar，就如我们所期望的。

so，请记住，少年。传给then方法的，一定要是一个函数哦。

## Solving the puzzle

上面所列出的让我们更加理解promise，那么，我们现在应该有能力去解决我在文首所提出的问题(puzzle).

下面是每个问题的解析，图文并茂哦:

#### Puzzle #1

```JavaScript
doSomething().then(function () {
  return doSomethingElse();
}).then(finalHandler);
```

Answer:

```
doSomething
|-----------------|
                  doSomethingElse(undefined)
                  |------------------|
                                     finalHandler(resultOfDoSomethingElse)
                                     |------------------|
```

#### Puzzle #2

```JavaScript
doSomething().then(function () {
  doSomethingElse();
}).then(finalHandler);
```

Answer:

```
doSomething
|-----------------|
                  doSomethingElse(undefined)
                  |------------------|
                  finalHandler(undefined)
                  |------------------|
```


#### Puzzle #3

```JavaScript
doSomething().then(doSomethingElse())
  .then(finalHandler);
```

Answer:

```
doSomething
|-----------------|
doSomethingElse(undefined)
|---------------------------------|
                  finalHandler(resultOfDoSomething)
                  |------------------|
```

#### Puzzle #4

```JavaScript
doSomething().then(doSomethingElse)
  .then(finalHandler);
```

```
doSomething
|-----------------|
                  doSomethingElse(resultOfDoSomething)
                  |------------------|
                                     finalHandler(resultOfDoSomethingElse)
                                     |------------------|
```

如果你依旧觉得这些个答案无法理解，那么我建议你重新阅读下这篇文章，抑或你可以在浏览器中自己尝试一篇。

## Final word about promises

promise还是非常不错的，如果你仍然在使用回调函数，那么我强烈建议你切换到promise，你的代码将会变得更加的轻量，更加的优雅，可读性也更高。

话虽如此说，但其实promise并不是完美的，它确实比回调函数要好，但这其实想在说: 一圈打到肚子上总比打在牙齿上要好。当然，如果你有更好的选择，你最后他们两个都不要用。

尽管比回调函数的方式要好，promise却仍然难以理解且容易让开发者犯错，这也是我写这篇博客的原因，新手个专家几乎都会碰到这些问题，同时我想强调的是，这并不是它们的错。这是promise的问题，promise和我们写同步代码的情景类似，但其实又不一样。

实际上，你原本不必要去学习一系列的晦涩的规则和新的API，在同步的世界里，你可以使用熟悉的return，throw，catch以及for等这些熟悉的东西来很好的写代码。同时，也不应该存在另一个类似的体系，来让你用同样的语句，却是不同的意义来编写代码。

