---
layout:     post
title:      "函数防抖与节流&lodash"
subtitle:   "debounce，throttle，JavaScript"
date:       2017-07-16 00:00
author:     "LanceLou"
header-img: "img/post-debounce-throttle-lodash.jpg"
catalog: true
tags:
    - JavaScript
---

> Lodash
> 
> A modern JavaScript utility library delivering modularity, performance & extras.
> 

Nodejs圈子比较有名的工具集，有非常多的npm包依赖于此。非常愉快的一个库，提供了JavaScript开发过程中常用到的一些工具函数，常见的is系列，to系列，copy等方法，当然还有我们今天的debounce，throttle。


## 区别

防抖(debounce)与节流(throttle)的区别是什么呢？这是一个很容易混淆的问题，进一步理解，你也会发现其实他们区别并不大。

> Throttling enforces a maximum number of times a function can be called over time. As in "execute this function at most once every 100 milliseconds."

-------------------

> Debouncing enforces that a function not be called again until a certain amount of time has passed without it being called. As in "execute this function only if 100 milliseconds have passed without it being called."

抓重点吗，节流指的是在一个时间段内，一个函数最多能被执行多少次。(every)

而防抖，则是指当函数被执行一次之后，过多久这个函数才能被再次执行。(after)

具体大家可以参考这里:

[The Difference Between Throttling and Debouncing](https://css-tricks.com/the-difference-between-throttling-and-debouncing/)

## 实现

这是我最想探讨的问题，显然，不管是debounce还是throttle，都是来限制函数被执行的次数，屌丝的实现可以是这样的:

```JavaScript
var timer = null;
function execFunction(func) {
  if (timer) {
  	clearTimeout(timer);
  	timer = null;
  }
  // 保证此处在1秒钟之内只被执行一次
  timer = setTimeout(func, 1000);
}
```

当然，上面只是一个屌丝版，同时也不失为抛砖引玉，lodash coming！


在Lodash中，throttle是基于debounce来实现的，即throttle直接调用了debounce函数，只是做了一些配置，所以我们从Lodash的debounce函数开始介绍。

需要看文件全概的同学可以戳[这里](https://github.com/lodash/lodash/blob/master/debounce.js)，在博文里我是分块讲的先。

首先是入口，自不必多说:

```JavaScript
function debounce(func, wait, options) {
  let lastArgs,
    lastThis,
    maxWait,
    result,
    timerId,
    lastCallTime

  let lastInvokeTime = 0
  let leading = false
  let maxing = false
  let trailing = true

  if (typeof func != 'function') {
    throw new TypeError('Expected a function')
  }
  wait = +wait || 0
  if (isObject(options)) {
    leading = !!options.leading
    maxing = 'maxWait' in options
    maxWait = maxing ? Math.max(+options.maxWait || 0, wait) : maxWait
    trailing = 'trailing' in options ? !!options.trailing : trailing
  }

  function debounced(...args) {
    const time = Date.now()
    const isInvoking = shouldInvoke(time)

    lastArgs = args
    lastThis = this
    lastCallTime = time

    if (isInvoking) {
      if (timerId === undefined) {
        return leadingEdge(lastCallTime)
      }
      if (maxing) {
        // Handle invocations in a tight loop.
        timerId = setTimeout(timerExpired, wait)
        return invokeFunc(lastCallTime)
      }
    }
    if (timerId === undefined) {
      timerId = setTimeout(timerExpired, wait)
    }
    return result
  }
  return debounced
}

export default debounce
```

这也是我们在一开始调用debounce函数时会做的一些初始化处理，有相关参数的初始化:

* lastInvokeTime: 最后一次源函数被调用的时间。
* lastCallTime: 最后一次debounce包装返回的函数被调用的时间。
* maxWait: 源函数被调用的最大delay时间，它确保在连续的debounce函数调用过程中，源函数能够在你给定的时间间隔内被连续调用。否则，在你整个debounce连续触发周期内，你的源函数只会累计被调用一次。
* leading ，trailing: timeout的结束时间还是开始时间，可以简单的理解为debounce内部是应该先调用源函数在开启timeout防抖还是先timeout，再在timeout之后才调用源函数。先等再掉还是先掉在等。


#### 基础实现

在lodash的debounce实现过程中，附加了如上给出的相关的一些特性，从而使得源码，嗯，这个，有点难读，其实我们可以从最基础的开始，再一步步进入加强版。

首先一个问题，lodash中是如何实现基本的debounce(在指定的时间段内对包装函数进行重复调用将会被忽略)。

其实就是一个timerId，其它的一切都是为其服务。

我们来看看精简版的debounced函数(在trailing阶段执行源函数)：

先看入口：

```
function debounced(...args) {
  const time = Date.now()
  const isInvoking = shouldInvoke(time)

  lastArgs = args
  lastThis = this
  lastCallTime = time

  if (isInvoking) {
    if (timerId === undefined) {
      // 防抖时间间隔已过。
      timerId = setTimeout(timerExpired, wait)
    }
  }
  return result
}
```

其实很简单，在shouldInvoke中判断是否能执行，在timeout后去真正执行源函数。

然后是shouldInvoke：

```
function shouldInvoke(time) {
  // 计算最后一次返回函数被调用的时间
  const timeSinceLastCall = time - lastCallTime

  return (lastCallTime === undefined || (timeSinceLastCall >= wait) ||
    (timeSinceLastCall < 0))
}
```

shouldInvoke函数被用来对比时间段，在基础款中，我们无需理会(maxing && timeSinceLastInvoke >= maxWait)。无非是看，咦，你不是防抖吗，你不是给了wait时间吗，wait时间到了没？到了就应该invoke了吧。

同时，我们还需要一个时间过期机制：也就是timerExpired函数，即经过指定的wait时间之后，打开开关，进行放行。

```JavaScript
function timerExpired() {
  const time = Date.now()
  if (shouldInvoke(time)) {
    return trailingEdge(time)
  }
  // Restart the timer.
  timerId = setTimeout(timerExpired, remainingWait(time))
}
```

这是一个非常有趣的函数，只要时间没到，通过setTimeout来保证自己能在真正过期的时候有相应的执行权。真正到达过期事件时，调用trailingEdge函数，trailingEdge做的事业很简单，清除timerId，调用源函数。此时我们其实可以看出，timerId为null时，表明我们的timerExpired已结束，新一轮的防抖已经开始啦。

#### 加强版(options.maxWait)

options.maxWait使用来干啥滴呢？在前面基础版中，我们的防抖函数会出现一种现象，即你进行持续触发时，即使过了指定的wait时间，也不会去触发源函数，因为我们每次调用debounced函数式，都会去更新我们的lastCallTime，这会使得shouldInvoke返回false。从而使得在timerExpired过后无法shouldInvoke通过去调用对应的源函数。即，你必须让你的debounced调用停下来wait的时间，再次调用才会调用源函数。

但有时，我们却是需要对应源函数是在特定的时间段之后被调用，而和我们的debounced函数无关。即保证源函数在特定的时间之后被invoke，当然前提是你调用了debounced函数。

这时就有我们的**options.maxWait**(The maximum time `func` is allowed to be delayed before it's invoked.)

其实这个机制主要的体现在于我们在shouldInvoke中加上了(maxing && timeSinceLastInvoke >= maxWait)，使用源函数被调用时间差来对比maxWait。

```JavaScript
function shouldInvoke(time) {
  const timeSinceLastCall = time - lastCallTime
  const timeSinceLastInvoke = time - lastInvokeTime

  // Either this is the first call, activity has stopped and we're at the
  // trailing edge, the system time has gone backwards and we're treating
  // it as the trailing edge, or we've hit the `maxWait` limit.
  return (lastCallTime === undefined || (timeSinceLastCall >= wait) ||
    (timeSinceLastCall < 0) || (maxing && timeSinceLastInvoke >= maxWait))
}
```

#### 加强版(leading 和 trailing)

leading 和 trailing，其实就是制定你是想执行源函数在进入防抖拒绝时间还是先过了这个时间在执行对应源函数。

这个的实现就有一系列的讲究了，分别有两个函数:

```
function leadingEdge(time) {
  // Reset any `maxWait` timer.
  lastInvokeTime = time
  // Start the timer for the trailing edge.
  timerId = setTimeout(timerExpired, wait)
  // Invoke the leading edge.
  return leading ? invokeFunc(time) : result
}
```
leadingEdge函数在已进入debounced函数时，如果符合相关条件，就会去调用。同时开启timerExpired


```
function trailingEdge(time) {
  timerId = undefined

  // Only invoke if we have `lastArgs` which means `func` has been
  // debounced at least once.
  if (trailing && lastArgs) {
    return invokeFunc(time)
  }
  lastArgs = lastThis = undefined
  return result
}
```

trailingEdge函数是在timerExpired函数之后调用的，也即指定时间段过后才去调用。

## throttle节流

```
function throttle(func, wait, options) {
  let leading = true
  let trailing = true

  if (typeof func != 'function') {
    throw new TypeError('Expected a function')
  }
  if (isObject(options)) {
    leading = 'leading' in options ? !!options.leading : leading
    trailing = 'trailing' in options ? !!options.trailing : trailing
  }
  return debounce(func, wait, {
    'leading': leading,
    'maxWait': wait,
    'trailing': trailing
  })
}
```

lodash中的节流函数简单，就是去调用其debounce函数，并显示指定maxWait就是wait。那，其实通过lodash中这种throttle的实现方式，***我们就可以粗略得出debounce和throttle的区别。debounce是外围调用级的函数执行限制，即限制call，而throttle是内在invoke级别的函数执行限制。防抖，是防止你频繁调用，强调外在调用，而节流，是放置函数执行次数过多，是内在的一种机制。***

## 总结

流，就像一根管子，不管你在外面如何塞，我也只能流过这么多，而防抖，则更像排队，前面的没有通过，你就不要硬是去挤，别人根本不理你。
