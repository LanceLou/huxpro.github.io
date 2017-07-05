---
layout:     post
title:      "middleware in redux"
subtitle:   "middleware redux koa"
date:       2017-07-02 00:00
author:     "LanceLou"
header-img: "img/post-middleware-in-redux.jpg"
catalog: true
tags:
    - ReactJS初体验
    - redux
    - koa
---

> 不忘初心，不负时光！
>
> ——LanceLou

好久不见，各位，自五月初入职某厂一来，一直沉迷于需求，无法自拔。体重增少许，肚子大少许，其他一切安好。尴尬减肥.jpg!

扯正题，说实话，在这段日子里，学到的挺多的东西，前端工程化，项目管理，团队协作等等。后期也会在这个博客中进行分享，总结。

如题，今天还是给各位谈谈这个中间件，啥是中间件，从传统意义上来说:***操作系统上，业务系统下与业务无关的 ，都是中间件。*** 脱离业务，连接'系统'！能够极大的减少业务层的开发成本。固然，在前端(FED)领域，中间件的概念基于此，却并未止于此。

## Redux middleware

It provides a third-part extension point between dispatching an action, and the moment it reaches the reducer.

在dispatch action与action到达reducer之间来监控，处理我们的action。和上文提到的传统意义上的middleware自有异曲同工之妙。

其作用也莫过于增强dispatch的功能。

当然，讨论middleware的作用并不是本文的主体，本文要讨论的，主要还是在redux中middleware的实现，其中所用到的函数是编程的思想与方式，是作者所看好了，也是react，redux生态所提倡的一种开发思想。

OK，废话不在多少，入正，正，正题。

## applyMiddleware

读读源码，其乐无穷！

```JavaScript
// code-1

import compose from './compose'

export default function applyMiddleware(...middlewares) {
  return (createStore) => (reducer, preloadedState, enhancer) => {
    const store = createStore(reducer, preloadedState, enhancer)
    let dispatch = store.dispatch
    let chain = []

    const middlewareAPI = {
      getState: store.getState,
      dispatch: (action) => dispatch(action)
    }
    chain = middlewares.map(middleware => middleware(middlewareAPI))
    dispatch = compose(...chain)(store.dispatch)

    return {
      ...store,
      dispatch
    }
  }
}
```

非常简短的一段applyMiddleware实现，那这个compose方法主要干了啥呢？其实这也是本文重点中的重点(敲黑板！)。

```JavaScript
// code-2

export default function compose(...funcs) {
  if (funcs.length === 0) {
    return arg => arg
  }

  if (funcs.length === 1) {
    return funcs[0]
  }

  return funcs.reduce((a, b) => (...args) => a(b(...args)))
}
```

#### compose

参数为funcs，自然需要接收一个方法的数组，compose顾名思义，组合，函数式编程中的组合。主要到这个语句：

```JavaScript
// code-3

return funcs.reduce((a, b) => (...args) => a(b(...args)))
```

Array.prototype.reduce，数组的一种聚合的方法，此处通过reduce，从函数a开始(funcs数组的第一个元素)，对后面出现函数进行逐个包装，将出现类似于下面的场景:

```JavaScript
// code-4

func -> [a, b, c, d];

first: (...args) => a(b(...args))
second: (...args) => first(c(...args)) 
third: (...args) => second(d(...args)) 
```

其实要理这种场景，我们只需要认识到一点，当我们这么去调用某个函数时:

```JavaScript
// code-5

someFunc1(someFunc2())
```

先执行的会是哪一个？ 我想说先执行的是someFunc2应该不会有人反对吧！执行循序，先求参数值再调用函数，这个没毛病！

OK，理解了这些，我们再去看code-3和code-4，无非是将函数进行层层包装，下一个下标的函数又会当做当前这个函数的参数并执行，同时其返回值也会当做当前函数的参数。是不是感觉中间件的场景若隐若现了？

#### createStore

介于我们是在createStore中来注入middleware，我们来结合createStore进行middleware的进一步讲解。

```JavaScript
// code-6
export default function createStore(reducer, preloadedState, enhancer) {
  // middleware作为preloadedState，将其转为enhancer
  if (typeof preloadedState === 'function' && typeof enhancer === 'undefined') {
    enhancer = preloadedState
    preloadedState = undefined
  }

  if (typeof enhancer !== 'undefined') {
    if (typeof enhancer !== 'function') {
      throw new Error('Expected the enhancer to be a function.')
    }
	 // 直接调用增强函数，将createStore对给对方。
    return enhancer(createStore)(reducer, preloadedState)
  }
  // 其实上面所做的不过是实现一种灵活的参数传递效果

  if (typeof reducer !== 'function') {
    throw new Error('Expected the reducer to be a function.')
  }

  // subscribe相关

  // dispatch相关挂载

  // replaceReducer......
}
```

如上，作者列出了createStore函数的一部分。可以看到，当我们需要通过传入参数来对我们的dispatch进行增强时，createStore直接会去调用增强函数，并将本函数传入。

其实此时，enhancer就是我们通过applyMiddleware包装出来的函数。此时，createStore又将createStore自身，以及reducer和initState(preloadedState)传递之。


```JavaScript
// code-7

import compose from './compose'

export default function applyMiddleware(...middlewares) {
//------- enhancer start
  return (createStore) => (reducer, preloadedState, enhancer) => {
    const store = createStore(reducer, preloadedState, enhancer)
    let dispatch = store.dispatch
    let chain = []

    const middlewareAPI = {
      getState: store.getState,
      dispatch: (action) => dispatch(action)
    }
    chain = middlewares.map(middleware => middleware(middlewareAPI))
    dispatch = compose(...chain)(store.dispatch)

    return {
      ...store,
      dispatch
    }
  }
//------- enhancer end
}
```

看我们的enhancer做了啥，createStore生成真正的store，这一次我们的createStore就会去将相关的方法挂载到store上去。开始包装middleware。

这个时候我们结合logger middleware进行讲解:

```
export default store => next => action => {
  console.log('dispatch: ', action);
  next(action);
  console.log('finish: ', action);
}
```

store参数由middlewares.map(middleware => middleware(middlewareAPI))进行传入。

那么next参数呢，

```
dispatch = compose(...chain)(store.dispatch)
```

这里compose组合函数其实本没有去调用我们的middleware函数，只是进行函数包装！真正对middleware进行第二次柯里化的是store.dispatch。当然，这个不会成为我们的next参数，在这之前，有一个非常magic的转化过程。先提出问题或者说现象:

1. 我们发现，在middleware中，next方法是可以阻断action在这个chain中的传播的，说的直接点，就是可以让action不传递给后继的middleware
2. 在我们讲解compose函数时，我们发现一个有趣的现象，照我们调用函数先算参数(先调用参数中的函数)的这个概念，最终的表现会从我们传入compose函数的funs的最后一个执行到第一个(循序不对)。

有这些问题很正常，因为咱们的源码还木有讲解完呀！

```
// 在compose(...chain)返回的包装好的函数之后，我们带上store.dispatch来执行一遍。
dispatch = compose(...chain)(store.dispatch)

// 然后我们结合compose返回组合完的函数的大概样子(大家可以带着compose的样子自行想象下)
function () {
  return a(b.apply(undefined, arguments));
}
```

这时候，真正magic的阶段来了！！！

我们将store.dispatch传入，这会成为b函数(chian中最后一个middleware)的参数。而其他的呢，以倒数第二个为例，将会接收最后一个middleware的返回进行第二次柯里化的参数(也就是咱们middleware中的next参数)传递:

```
// wrappedFunc: 最后一个中间件接收到store.dispatch的返回，咱们暂且叫他wrappedFunc
action => {
  // 中间件真正逻辑
}

倒数第二个middleware执行前
next => action => {
  console.log('dispatch: ', action);
  next(action);
  console.log('finish: ', action);
}
// 这时wrappedFunc 就成了这里的next了，执行后
action => {
  console.log('dispatch: ', action);
  next(action); // next即wrappedFunc
  console.log('finish: ', action);
}
```
以此类推，直至我们的chain中的第一个middleware。我们发现，顺者进行一遍chain的包装，然后逆着又进行一遍next参数的'注入'，返回第一个middleware的逻辑，并"注入"next参数。返本还源。最终挂载在store上的dispatch其实就是怎们两次包装完的第一个middleware，redux作者还是非常OK的。

最终，我们的middleware函数只需要一个action参数了，这不就是我们的dispatch吗(需要一个action参数)，enhancer其实就是增强dispatch。

#### 小结

两次middleware chain的柯里化参数包装(传入)，1个是传入middlewareAPI(store)，第二次是compose对middleware chian进行串联后调用串联后的结果进行next参数的反序传入。middleware串联的效果，出来的毫无违和感！

## Middleware Ext

其实，说道middleware，我们不得不提另一个和咱们Feder'亲密'框架，或者我们其实也不能叫它框架。那就是Koa。

![middleware graph](https://www.lancelou.com/img/in-post/koa-middleware.jpeg)

Koa的中间件技术和Redux其实是大同小异的，请求和响应两个箭头其实就是next()调用的上面和下面的部分。blabla！

## 总结

middleware模式其实是一种非常好的系统编程的模式，连接业务与'系统'，其实说道这种分层思想，学过计算机网络的小伙伴应该不会陌生。各层之间相互合作，层层信息的打包与up package，各层之间相互合作又相互透明(独立与隔离)。进一步又有热插拔等等。当然，这些都扯远了，但谁又能否认这些知识之间的千丝万缕的联系呢！

是吧，老铁！

