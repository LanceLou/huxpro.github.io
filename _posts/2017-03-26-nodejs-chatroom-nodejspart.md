---
layout:     post
title:      "socket聊天室-NodeJS篇"
subtitle:   "socket.io express MongoDB-sessionStore SPA"
date:       2017-03-26 00:00
author:     "LanceLou"
header-img: "img/post-nodejs-chartroom-nodejspart.jpg"
catalog: true
tags:
    - NodeJS初体验
---

> 因上努力，果上随缘

基于socket.io构建的一个在线的聊天室，也是参考教程一步一步做出来的，这里浏览器端暂时用的是angularJS，后期作者将会敲一个React版的出来。

其实作者学习NodeJS时间不长，实战方便较为缺乏，此次通过这一个小小的聊天室的项目学习，对整个NodeJs的项目的搭建也算有了个浅显的了解吧。

## WebSocket

浏览器与服务器实时(real time)交互的关键技术(双工通信)，通过这项技术，在同一个websocket连接上，浏览器可以向服务器发送消息，服务器也可以向浏览器发送消息。打破了传统的请求应答模式。

#### socket.io

浏览器与服务端(NodeJS)实时通信的一个lib，且提供跨终端的支持，其在实时通信方面提供优雅降级的解决方案。从高到底依次为:

* Websocket
* Adobe® Flash® Socket
* AJAX long polling
* AJAX multipart streaming
* Forever Iframe
* JSONP Polling

#### Adobe® Flash® Socket

基于浏览器插件的实时通信，这个笔者了解不多，这几年Flash也有唱衰的趋势，此处暂且不聊，有需要的读者可以自行Google。

#### AJAX long polling

ajax长轮询，这个很容易理解，怎么说呢，我觉得这是一种伪实时通信，通过ajax的持续询问(query)，来获取服务端实时的状态数据更改。

具体实现也不困难，在ajax通信结束的时候(xhr.readyState == 4, done), 再次新建一个xhr。

```JavaScript
var longPoll = function(type, url){
    var xhr = new XMLHttpRequest();

    xhr.onreadystatechange = function(){
        // 状态为 4，数据传输完毕，重新连接
        if(xhr.readyState == 4) {
            receive(xhr.responseText);
            xhr.onreadystatechange = null;

            longPoll(type, url);
        }
    };

    xhr.open(type, url, true);
    xhr.send();
}
```

#### AJAX multipart streaming

数据流方式，在建立的连接断开之前，也就是 readystate 状态为 3 的时候接受数据。此时是loading状态，链接是一直不会断开的，而此时server的数据是分多部分发送过来的(multipart), 在loading过程中，server又可以将状态变化发送给客服端。

```
var dataStream = function(type, url){
    var xhr = new XMLHttpRequest();

    xhr.onreadystatechange = function(){

        // 状态为 3，数据接收中，采用游标记录数据位置，从而判定传输的完成
        if(xhr.readyState == 3) {
            var i, l, s;

            s = xhr.response; //读取数据
            l = s.length;     //获取数据长度

            //从游标位置开始获取数据，并用分割数据
            s = s.slice(p, l - 1).split(splitChar);

            //循环并操作数据
            for(i in s) if(s[i])  deal(s[i]);

            p = l;  //更新游标位置

        }

        // 状态为 4，数据传输完毕，重新连接
        if(xhr.readyState == 4) {
            xhr.onreadystatechange = null;

            dataStream(type, url);
        }
    };

    xhr.open(type, url, true);
    xhr.send();
};
```

#### Forever Iframe

利用iframe来获取数据，不过是Forever的，Forever的原因就是real time的原因。iframe也算是浏览器支持性最好的特性。在xhr没有出现之前，在页面中与浏览器进行通信，iframe是一个不错的hack。当然，还有我们下面提到的JSONP。

#### JSONP Polling

JSONP + Polling， polling的原因不多说，JSONP本来是一种跨域解决方案，通过script标签来获取服务端更新，通过回调函数的参数来传递数据。

## express

***router模块***: NodeJS异步流程控制库，前身为connect，NodeJS以及整个JavaScript生态的异步性我想咱们都是没有异议的。

#### 何为异步流程控制

何为异步流程控制，我们的NodeJS服务器在处理请求的时候，咱们抛开应用层以下的NodeJS底层以及OS底层对数据的封装以及打包，成帧什么的，最终到我们应用层Node能够拿到的，也是一串字符不是，这里面就有Http协议头，请求参数或数据，cookie( -> session)，数据加密等，自然我们是要从这里面提取数据的。http请求req -> data/cookie -> [解密] -> session。后一个处理依赖于前一个处理，同时你也可以操作其中任意项。

当然你说我写一个函数，你要什么我就从里面给你提取出什么，但这中耦合性太高，工程化效率低。

比较好的处理方式就是针对每一个我们需要提取的数据，我们编写一个函数，将结果传给下一个函数，下一个函数再处理。就像流水线作业一样；我觉得这不仅符合工程化的流水线思想，同时也符合计算机网络的分层处理思想的。

这个多函数，这么多流水线上的节点，如何去让他们有循序跑起来，并且是 ***异步*** 的跑起来。这就是流程，异步流程。这里跳过 **回调地狱** 的描述，直接说说express中异步流程处理的思想。

#### middlewares+next

中间件就是我们上面提到的各种function， 处理自己对应的本分的工作，如cookie-parser，body-parser等，而next就是流程控制的关键，在处理完本轮的工作之后，去调用流水线中的下一个节点。直至得到所需要的数据，或者说某一个特定的function。

当然，express会去维护一个middlewares的stack来存储。

app.use就是向这个stack中push进中间件。

```
// express 配置中间件
app.use(bodyParser.urlencoded({
  extended: true
}));
app.use(bodyParser.json());
app.use(cookieParser())
app.use(session({
  secret: 'technode', //加密签名
  cookie:{
    maxAge: 60 * 1000
  },
  ttl: 14 * 24 * 60 * 60,
  store: sessionStore,
  resave: true, //resave : 是指每次请求都重新设置session cookie，假设你的cookie是10分钟过期，每次请求都会再设置10分钟
  saveUninitialized: true //是指无论有没有session cookie，每次请求都设置个session cookie ，默认给个标示为 connect.sid
}))
```

当然，这只是异步流程流程控制中的一种场景。其实在NodeJS web服务器中，异步流程存在的还是相当多的。同时衍生了像async.js这样优秀的异步流程控制库。后期作者要着重介绍之。

***express中其他模块***

request，response封装的一个可读一个可写对象，web服务器必备。

application: 配置对象。

## SPA

这里作者浅谈一下SPA，单页面应用，其实就是把router放到了浏览器，然后通过一系列的ajax来无刷新的与服务端进行交互。



## 文件与模块划分

```
├── controllers (控制器，执行数据的增删改查与组装)
│  ├── messages.js
│  ├── rooms.js
│  ├── users.js
│  ├── index.js
│
├── models (models数据库描述文件)
│  ├── messages.js
│  ├── room.js
│  ├── user.js
│  ├── indes.js
│
├── node_modules
│  ├── ...
│
├── services (业务处理，服务接口handler，api)
│  ├── api.js
│  ├── socketApi.js
│
├── static
│  ├── 前端静态文件
│
├── app.js (main，入口文件，服务器配置，接口监听，验证授权等)
├── config.js (server配置文件)
├── package.json
```

对应的功能在上面图中有说明，这里说一下models数据描述文件，里面存的就是Schema，这是mongoose的数据描述文件，因为mongo是非关系型数据库，不存在表这种东西，所以我们通过Schema来对数据库存的数据进行类似建模。从而方便mongoose进行数据库的增删改查工作。

```
var mongoose = require('mongoose')
var Schema = mongoose.Schema,
  ObjectId = Schema.ObjectId

var Message = new Schema({
  content: String,
  creator: {
    _id: ObjectId,
    email: String,
    name: String,
    avatarUrl: String
  },
  _roomId: ObjectId,
  createAt:{type: Date, default: Date.now}
})

module.exports = Message
```

## 结语

在线聊天室的服务端的总结在这就告一段落了，读者也可以移步我的github查看跟多细节的部分，当然，项目还有许多的不足，木有写测试用例，木有不是自动化组件，木有打包等，但毕竟刚起步不是，慢慢来，慢慢沉淀。

## Refers

* [technode](https://github.com/island205/technode-tutorial)
* [JavaScript之web通信](http://www.cnblogs.com/hustskyking/p/web-communication.html)
* [《Node.js 包教不包会》 by alsotang](https://github.com/alsotang/node-lessons/tree/master/)

—— The end

—— 2017.03 LanceLou