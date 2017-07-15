---
layout:     post
title:      "HTTP catch(Expiration Model + Validation)，CORS之Preflighted"
subtitle:   "HTTP，CORS，JavaScript"
date:       2017-07-10 00:00
author:     "LanceLou"
header-img: "img/post-http-catch-js-cors.jpg"
catalog: true
tags:
    - HTTP
    - JavaScript
---

> 他寻求着什么，在遥远的异地！
> 
> 他抛下什么，在可爱的故乡！
> 
> -- <<帆>> 节选
>

OK，话题切换，今夜，我们来谈谈HTTP协议，顺带提一下CORS，也算是前后台交互相关(源)，就不另起篇幅了。

## 起因

最近在做管理系统相关项目时，出现模板HTML更改上线后，用户那边报错，清下缓存即解决。当然，在线上发布时，我们的JavaScript和css相关资源是有打版本号的(gulp-rev，gulp-rev-replace，gulp-useref)。但在用angular的路由时，其请求的template html是由angular内部去维护的，其实就是使用ajax去请求对应的模板文件。因为我们并没有给这些路由对应的template加上版本号，故在我们有对相关html进行更新时，客服端依旧使用浏览器本地缓存，这是表现。有关于angular路由缓存的问题不是本文的重点(这里有一个简短的讨论: [Stop AngularJS from caching html between routes
](https://stackoverflow.com/questions/31096529/stop-angularjs-from-caching-html-between-routes))。

主要还是在查看Chrome的Network面板的是否发现用请求是 200 (from cache)。话说明眼人一看这不from cache嘛，缓存啊，那和304区别是啥呢？

而CORS的Preflighted，场景我是忘了，也是从控制台的报错里面发现这个知识点的。

OK。入正题！

## HTTP Cache

**Caching would be useless if it did not significantly improve performance.**

老生常谈的问题，至于为什么用cache，自然每个人有每个人的观点，此处我只想带着规范，来好好读下这个HTTP Cache。

> The former reduces the number of network round-trips required for many operations; we use an "expiration" mechanism for this purpose (see section 13.2). The latter reduces network bandwidth requirements; we use a "validation" mechanism for this purpose (see section 13.3).

这就是我们今儿要弹的两个主角儿，expiration机制和validation机制。通过expiration机制，我们可以直接减少网络通信round-trips(一来一回)的次数，而通过validation机制，我们可以减少http交互所占带宽(数据回传大小)。

说白了，expiration机制决定是要不要和后端交互，validation机制决定请求数据要不要重新发送给客服端。

#### Expiration Model(Freshness)

HTTP caching最佳(best)的工作状态： 连给服务器的请求都不用发。服务器提供一个过期时间，在这个时间点之前，当前的response是满足客服端接下来的请求的(可复用)，即咱们可以默认从缓存里取的就是最新的。

服务端可以通过Expires header，或者Cache-Control header中的max-age来让浏览器进入这种模式，二者取一即可，但二者的区别不是本文所讨论的。

Expiration Model如果浏览器"验证请求通过"，就会从缓存中直接取resposne，也就是我们见到的200(from cache)。

当然，这是一种投机，或者说需要强大保证才能进行的机制，你得保证在特定时间内，你的数据不会进行更新。类似于地址库，一些常用图标资源文件等。


#### Validation Model

这一阶段需要服务端的参与，Validation的，验证验证，就得找资源的主人来验证下。

> 歪，老铁，上次你给我的那个饼干还能吃吗？我看保质期上面的保质期都已经过了。

在这个模式下，我们的客服端会向服务端发送一个请求来check对应资源是否可直接从本地浏览器缓存中取。这就是validating。这能保证在validate本地资源成功的情况下，我们就不必去重传完整的response。同时，其实即使是资源已过期，通过Http的conditional methods，我们也无需去建立额外的http连接来获取新的资源。

> HTTP conditional requests are requests that are executed differently, depending on the value of specific headers. These headers define a precondition, and the result of the request will be different if the precondition is matched or not.

在Validation模式下，我们还需要去添加一些"cache validators."，也可称之为缓存验证因子。这些validators是在上一次完整的服务端返回(即缓存验证失败，服务端返回完整的资源)时被顺便带回来的。自此，这些个validators就会一直跟着对应的资源，直至下一次服务端完整的返回就会被更新。客服端再次发起对应资源的请求时，就会带上validators。服务端根据这些validators进行前后对比，匹配未更改-> 304, 否则，还用问？资源给我完整返回哈！

***cache validators***，也即是我们常见的Last-Modified以及Entity Tag，Last-Modified，最后被修改时间，这个其实只要两边对不上，铁定已修改。而Entity Tag是根据文件内容生成的hash值。有两组字段的原因是Last-Modified是HTTP 1.0协议所引入的，但由于担心前后端时钟不一致以及Last-Modified是只有秒级精确，如果在一秒内有一次以上的文件修改，就会产生问题。(对比上面的Expiration和Cache-Control，其实也是一个意思)。

***Weak and Strong Validators***

这里还有提到强验证和弱验证，可一并作下对比。

Strong Validators: if the entity (the entity-body or any entity- headers) changes in any way, then the associated validator would change as well.

Weak Validators: A validator that does not always change when the resource changes


## CORS

跨域资源共享，这个自不必多说，其实本文也只是提它的一个细节点，跨域资源共享标准规范要求: 对那些可能对服务器数据产生副作用的 HTTP 请求方法（特别是 GET 以外的 HTTP 请求，或者搭配某些 MIME 类型的 POST 请求），浏览器必须首先使用 OPTIONS 方法发起一个预检请求（preflight request），从而获知服务端是否允许该跨域请求。服务器确认允许之后，才发起实际的 HTTP 请求。在预检请求的返回中，服务器端也可以通知客户端，是否需要携带身份凭证（包括 Cookies 和 HTTP 认证相关数据）。

这就是preflight request，预检测，与服务器沟通是否允许跨域。

## 词汇解释

***在阅读http的rfc2616文档时，作者遇到了一些比较难以理解的词汇，这里作下汇总：***

1. Semantic transparency(语义的透明): Semantic transparency is a descriptive phrase that has been used in linguistics to describe endocentric compounds(向心复合词). Endocentric compound words are those whose whole meaning can be figured out by an analysis of its parts or "morphemes". An example of an endocentric compound is the word "car-wash". 我们可以根据这个词的构成，来判断它的语义。Http缓存的语义透明，其所表明的是缓存不会改变客服端与服务器端的请求响应之间交换的意义，对整体的HTTP语义不会造成太大影响，介入性不会那么高！

## 总结

其实像HTTP缓存以及诸如CORS都是特别常见的知识点，但其实有些细节的地方是我们没有去注意的，这些东西可能平时开发过程中用不到，或者说即使你不知道也没有太大关系，但在解决一些关键性的问题时，这些知识或许会起到意想不到的作用。多观察，多思考，多总结！！！！！！

## refers

1. [Caching in HTTP](https://www.w3.org/Protocols/rfc2616/rfc2616-sec13.html)
2. [HTTP访问控制（CORS)](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Access_control_CORS)