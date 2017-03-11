---
layout:     post
title:      "NodeJS初体验-tick与eventloop"
subtitle:   "nextTick, setTimeout, NodeJS运行机制"
date:       2017-03-08 00:00
author:     "LanceLou"
header-img: "img/post-node-eventloop-nextTick.png"
catalog: true
tags:
    - NodeJS初体验
---

## 前言

[查看大图(作者的一个简单的思考过程)](http://og4j2atko.bkt.clouddn.com/Mind-node-tick-b.png)

最近作者在学习nodejs，应社区的大佬的指导，去看了官方的文档，说实话，还是有那么一些吃力的。在作者的这几天的学习过程中，些许有趣的经历，拿来和大家分享下。

在promise这一节，发现了process.nextTick(callback[, ...args])这么个有趣的函数。doc这么说:

>The process.nextTick() method adds the callback to the "next tick queue". Once the current turn of the event loop turn runs to completion, all callbacks currently in the next tick queue will be called.

> This is not a simple alias to setTimeout(fn, 0). It is much more efficient. It runs before any additional I/O events (including timers) fire in subsequent ticks of the event loop.


啊哈，next tick queue, event loop, ***more efficient!!!*** 始于这几个关键词，Lance的NodeJS奇趣之旅就此开始，呜呜呜，快上车。

## 发现

为什么nextTick要比setTimeout高效(more efficient)? 这个问题我们放到文末在解释。

event loop, tick queue, 这个好解释，event loop咱们在JavaScript也有，那么说道NodeJS中的这个event loop，逃不开一个底层库libev(现集win平台的相关系统调用改名为libuv -> 跨平台)。它为 Node.js 提供了跨平台，线程池，事件池，异步 I/O 等能力，自然也是event loop的源泉(异步, 事件机制)。

![nodejs](https://www.tuchuang001.com/image/urB8a)

#### libuv

> libuv is cross-platform support library which was originally written for NodeJS. It’s designed around the event-driven asynchronous I/O model.

专为NodeJS写的，事件驱动和异步IO，跨平台的lib。调用系统I/O，网络。异步回调handler。

![libuv](https://www.tuchuang001.com/image/urxvj)

那么事件驱动，异步I/O，自然需要一个机制，去听，去看哪个事件发生了，哪个系统调用有了响应，有了就赶紧回调，但你也别再哪儿守株待兔。

#### event loop

下图展示了event loop的操作顺序

```

   ┌───────────────────────┐
┌─>│        timers         │
│  └──────────┬────────────┘
│  ┌──────────┴────────────┐
│  │     I/O callbacks     │
│  └──────────┬────────────┘
│  ┌──────────┴────────────┐
│  │     idle, prepare     │
│  └──────────┬────────────┘      ┌───────────────┐
│  ┌──────────┴────────────┐      │   incoming:   │
│  │         poll          │<─────┤  connections, │
│  └──────────┬────────────┘      │   data, etc.  │
│  ┌──────────┴────────────┐      └───────────────┘
│  │        check          │
│  └──────────┬────────────┘
│  ┌──────────┴────────────┐
└──┤    close callbacks    │
   └───────────────────────┘
```

****note: each box will be referred to as a "phase" of the event loop.(图中每一个矩形都代表event loop的一个时期， 本节的讨论也是以一个phase为基本单位)****

>Each phase has a FIFO queue of callbacks to execute. While each phase is special in its own way, generally, when the event loop enters a given phase, it will perform any operations specific to that phase, then execute callbacks in that phase's queue until the queue has been exhausted or the maximum number of callbacks has executed. When the queue has been exhausted or the callback limit is reached, the event loop will move to the next phase, and so on.

>Since any of these operations may schedule more operations and new events processed in the poll phase are queued by the kernel, poll events can be queued while polling events are being processed. As a result, long running callbacks can allow the poll phase to run much longer than a timer's threshold. See the timers and poll sections for more details.(在轮询(poll)阶段，由于其它各个阶段的操作都有可能导致新的事件发生并使得内核向轮询阶段的queue中添加事件，所以在poll阶段处理事件的时候可能还会有新的事件产生，最终，长时间的调用回调函数将会导致定时器过期，所以在poll阶段与定时器会有"合作", 这个在下面会提到。)。

各个时期

* timers: this phase executes callbacks scheduled by setTimeout() and setInterval().执行过时定时器的回调
* I/O callbacks: executes almost all callbacks with the exception of close callbacks, the ones scheduled by timers, and setImmediate().

This phase executes callbacks for some system operations such as types of TCP errors. For example if a TCP socket receives ECONNREFUSED when attempting to connect, some *nix systems want to wait to report the error. This will be queued to execute in the I/O callbacks phase.

* idle, prepare: only used internally.(NodeJS 内部使用)
* poll: retrieve new I/O events; node will block here when appropriate.(轮询I/O事件，响应系统调用反馈，处理handler，block here when appropriate)。
* check: setImmediate() callbacks are invoked here.处理setImmediate的回调函数
* close callbacks: e.g. socket.on('close', ...).执行close cb handler。

***注意上面六个阶段都不包括 process.nextTick()***

这里着重讲一下poll阶段:

poll phase main funciton:

1. Executing scripts for timers whose threshold has elapsed, (执行过期的定时器，这里不要被表意所迷惑，所谓Executing 是指跳到对应的phase)then
2. Processing events in the poll queue.(处理queue中的cb)

>When the event loop enters the poll phase and there are no timers scheduled, one of two things will happen:（没设置定时器）
>
> ->***If the poll queue is not empty***, the event loop will iterate through its queue of callbacks executing them synchronously until either the queue has been exhausted, or the system-dependent hard limit is reached.
>
> ->***If the poll queue is empty***, one of two more things will happen:poll queue没有回调函数
>
> ->***If scripts have been scheduled by setImmediate()***, the event loop will end the poll phase and continue to the check phase to execute those scheduled scripts.有schedule setImmediate，马上跳到check阶段去处理。
>
> ->***If scripts have not been scheduled by setImmediate()***, the event loop will wait for callbacks to be added to the queue, then execute them immediately.没有schedule setImmediate，阻塞等待。
>
> 有设置定时器：
>Once the poll queue is empty the event loop will check for timers whose time thresholds have been reached. If one or more timers are ready, the event loop will wrap back to the timers phase to execute those timers' callbacks.


其实我们去看源码的话，会发现，event loop就是由下面的uv_run发起的
[https://github.com/nodejs/node/blob/v6.x/deps/uv/src/unix/core.c](https://github.com/nodejs/node/blob/v6.x/deps/uv/src/unix/core.c)

```
int uv_run(uv_loop_t* loop, uv_run_mode mode) {
  int timeout;
  int r;
  int ran_pending;

  r = uv__loop_alive(loop);
  if (!r)
    uv__update_time(loop);

  while (r != 0 && loop->stop_flag == 0) {
    uv__update_time(loop);
    uv__run_timers(loop);
    ran_pending = uv__run_pending(loop);
    uv__run_idle(loop);
    uv__run_prepare(loop);

    timeout = 0;
    if ((mode == UV_RUN_ONCE && !ran_pending) || mode == UV_RUN_DEFAULT)
      timeout = uv_backend_timeout(loop);
	
    uv__io_poll(loop, timeout); //开启轮询，传入了timeout。这个是关键，也就是我们上文提到的poll phrase有定时器的话的处理。
    uv__run_check(loop);
    uv__run_closing_handles(loop);

    if (mode == UV_RUN_ONCE) {
      /* UV_RUN_ONCE implies forward progress: at least one callback must have
       * been invoked when it returns. uv__io_poll() can return without doing
       * I/O (meaning: no callbacks) when its timeout expires - which means we
       * have pending timers that satisfy the forward progress constraint.
       *
       * UV_RUN_NOWAIT makes no guarantees about progress so it's omitted from
       * the check.
       * 
       * 上面这句话很重要: UV_RUN_ONCE这种模式啊，代表着必定有进一步的"发展"(理解为状态在
       * 往前发展，有改变)，为什么呢，因为当他返回时至少会有一个回调函数被调用。即执行
       * uv__io_poll时，如果有rb，自然会执行rb，如果没有；其就会在定时器过期之前return，这
       * 就意味着已近有正在等待的定时器可以执行进一步的操作。wrap back to the timers
       * phase。
       * 
       * 而如果是UV_RUN_NOWAIT，这是非阻塞的，直接调用系统的no wait机制，它的返回，是不保
       * 证有进一步progress的！！！！！！
       */
      uv__update_time(loop);
      uv__run_timers(loop);
    }

    r = uv__loop_alive(loop);
    if (mode == UV_RUN_ONCE || mode == UV_RUN_NOWAIT)
      break;
  }

  /* The if statement lets gcc compile it to a conditional store. Avoids
   * dirtying a cache line.
   */
  if (loop->stop_flag != 0)
    loop->stop_flag = 0;

  return r;
}
```

#### process.nextTick()

> process.nextTick() is not technically part of the event loop. Instead, the nextTickQueue will be processed after the current operation completes, regardless of the current phase of the event loop. 不管当前event loop的时期，只要当前的操作一完成，就会去处理nextTickQueue。这个是如何实现的作者哈还需要去好好理解，不过大概可理解为肯定能在下一个phrase给你处理nextTickQueue。

同时也指出:

> Looking back at our diagram, any time you call process.nextTick() in a given phase, all callbacks passed to process.nextTick() will be resolved before the event loop continues. This can create some bad situations because it allows you to "starve" your I/O by making recursive process.nextTick() calls, which prevents the event loop from reaching the poll phase. 不要递归调用，否则会阻塞整个event loop！！！

为什么NodeJS要设计这个API呢？

> Why would something like this be included in Node.js? Part of it is a design philosophy where an API should always be asynchronous even where it doesn't have to be. 为什么NodeJS中要有这个nextTick()，我也很奇怪，这里回答了，一种设计哲学~~(思维)，即使一个API没必要异步，我也要把他设计成异步。即使一个操作没有必要异步，我也要把他异步了。异步好啊，搞完事儿就走，自认会有人来处理。

举个例子:

```js
function apiCall (arg, callback) {
  if (typeof arg !== 'string') //API调用不合规，将错误传入回调函数，而不是立即报错。
    return process.nextTick(callback,
      new TypeError('argument should be string'));
}
```

这是一个很好的例子，相信写NodeJS比较多的同学都知道，NodeJS常见的异步调用方式是不是这样:

```js
const fs = require('fs');

fs.unlink('/tmp/hello', (err) => {
  if (err) throw err;
  console.log('successfully deleted /tmp/hello');
});
```

这样的话可以说很符合异步的规范，并且我们可以保证回调函数会执行(By using process.nextTick() we guarantee that apiCall() always runs its callback after the rest of the user's code and before the event loop is allowed to proceed.)。其实就是，我很快调用，但是是异步调用。

```js
function someAsyncApiCall (callback) {
  process.nextTick(callback); //此处如果直接 callback()的话，那就是同步调用了，这也是
  //nextTick再次的作用，为什么要这样，注意bar变量，the script still has the ability 
  //to run to completion, allowing all the variables, functions
};

someAsyncApiCall(() => {
  console.log('bar', bar); // 1
});

var bar = 1;

```
同样: 

```js
const server = net.createServer(() => {}).listen(8080);
//当我们初始化port监听的时候，咱们的on('listening')还没执行呢！！！
server.on('listening', () => {console.log("listening -> on")});
```

然鹅，显然是，有执行(这个大家都懂)，

>When only a port is passed the port is bound immediately. So the 'listening' callback could be called immediately. Problem is that the .on('listening') will not have been set by that time.
>
>To get around this the 'listening' event is queued in a nextTick() to allow the script to run to completion. Which allows the user to set any event handlers they want.  
>-> 'listening' event是在nextTick里去添加到queue中的。所以在这之前，显然on是执行了的。


## process.nextTick() & setImmediate() & setTimeout()

***前面有了这些解释，咱们入正题。***

#### process.nextTick() vs setImmediate()

* process.nextTick() fires immediately on the same phase(phase结束完立即调用)
* setImmediate() fires on the following iteration or 'tick' of the event loop -> event loop的check阶段执行。

-> **process.nextTick() fires more immediately than setImmediate()**

#### process.nextTick() vs setTimeout()

这个就木有可比性了，process.nextTick每个phrase都会去查看queue并执行cb。而Timer超时是在event loop开始和结束的时候check。

#### setImmediate() vs setTimeout()

```
setTimeout(function timeout () {
  console.log('timeout');
},0); //注意这儿的第二个参数

setImmediate(function immediate () {
  console.log('immediate');
});
```

分两种情况，如果在上面的代码在回调中执行(比如I/O事件的cb)，那就好办了: 

I/O的cb在uv_io_poll phrase执行，那么从uv_io_poll中出来之后，马上到了check phrase，所以Immediate事件被立即执行。

最后在event loop的结束阶段检查超时，执行对应回调。

-> 如果是在非回调函数中执行:那么这两个回调的执行顺序是不定的，大家可以试下

Why？

咱们是不是已经了解了这两个执行所在的phrase或者说"位置", setImmediate的cb在check phrase执行，而setTimeout对应的Timer会在event loop的开始和结束位置检查。那按理来说应该一开始就执行了的，其实不全是。

> 在node中，setTimeout(cb, 0) === setTimeout(cb, 1);
> 
> 确实每次loop进来，都是先检查uv_run_timer的，但是由于cpu工作耗费时间，比如第一次获取的hrtime为0, 那么setTimeout(cb, 1)，超时时间就是loop->time = 1(ms，node定时器精确到1ms，但是hrtime是精确到纳秒级别的)
所以第一次loop进来的时候就有两种情况：
>
>1.由于第一次loop前的准备耗时超过1ms，当前的loop->time >=1 ，则uv_run_timer生效，timeout先执行
>
>2.由于第一次loop前的准备耗时小于1ms，当前的loop->time = 0，则本次loop中的第一次uv_run_timer不生效，那么io_poll后先执行uv_run_check，即immediate先执行，然后等close cb执行完后，继续执行uv_run_timer
>

## 总结与感想

笔者接触NodeJS时间很短，最近在瞅NodeJS的文档，看文档可能比较慢。但其实从文档中发散出来的知识点，给了笔者一个很大，很高的角度去理解整个NodeJS的系统(平台)，以及与其先关的社区，技术迭代。

JavaScript是真单线程，而NodeJS的JavaScript部分确实还是单线程，但整个平台，其I/O，是多线程去跑的。

回调自然是好，但回调最终的cb还是由JavaScript的主线程去处理，回调是I/O，计算(用户代码JavaScript的执行)还是主线程在做。所以说咱们的NodeJS啊，确实是适合网络，文件读写这类高I/O的工作，且性能高。

而不适合的，自然是像加密解密，图形图像处理这类CPU密集型处理的工作。

## Refer

* [The Node.js Event Loop, Timers, and process.nextTick()](https://github.com/nodejs/node/blob/v6.x/doc/topics/event-loop-timers-and-nexttick.md)
* [Node.js Event Loop 的理解 Timers，process.nextTick()](https://cnodejs.org/topic/57d68794cb6f605d360105bf)
* [神奇的nextTick
](http://www.cnblogs.com/lengyuhong/archive/2013/03/31/2987745.html)
* [Node.js 探秘：初识单线程的 Node.js
](http://taobaofed.org/blog/2015/10/29/deep-into-node-1/)
* [Welcome to the libuv API documentation](http://docs.libuv.org/en/v1.x/index.html)
* [libev 设计分析](https://cnodejs.org/topic/4f16442ccae1f4aa270010a3)
* [nodejs 异步之 Timer &Tick; 篇](https://cnodejs.org/topic/4f16442ccae1f4aa2700109b)
