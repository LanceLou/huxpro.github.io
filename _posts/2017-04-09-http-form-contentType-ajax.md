---
layout:     post
title:      "JavaScript ajax表单模拟与koa处理"
subtitle:   "http content-type ajax koa"
date:       2017-04-06 00:00
author:     "LanceLou"
header-img: "img/post-http-form-contentType-ajax.jpg"
catalog: true
tags:
    - JavaScript
    - HTTP
---

> Houston, we have a problem. 
> 
> —— Apollo 13

以前写ajax，由于参数不多，数据传输不大，都给写request params里面了，文件上传呢都是直接去读取流去可以了，毕竟有Web容器，底层库(Servlet)给我们封装了，我们只要安心处理路由，业务即可。从来就没有去考虑将数据放在request的body里(表单上传)，以及通过何种上传的content-type来经过不同的解析从而获得最终的数据。

但当你碰见了NodeJS，那就不一样了，其实也不是说不一样，也看你用不用对应的中间件，他们会自动给你parse，但几乎没有一个中间件，或者是我没发现，会像javaweb 中的Servlet一样，帮你做很多很多的事情。也许这就是Node生态的组件化，模块化（Package），高定制性。就像前几天跟着一个比较老的教程学习Express，里面用的是Express3，发现在新版的Express4中，移除了许多的中间件。


Express 3 | Express4
:---- | :---
express.bodyParser | body-parser + multer
express.compress |  compression
express.cookieSession |  cookie-session
express.cookieParser |  cookie-parser
express.logger |  morgan
express.session |  express-session
express.favicon |  serve-favicon
express.responseTime |  response-time

等等等等等......(未完)

左边对应的是之前Express3 build-in的中间件，右面是对应的独立出来的中间件。详情可访问 [Moving to Express 4](https://expressjs.com/en/guide/migrating-4.html)

OK 扯远了，静瞎逼逼。


## 表单数据上传，XMLHttpRequest

这个前端er似绝对不会陌生的，我们一般的页面无刷新上传就是用的它，XMLHttpRequest的表单上传有两种方式:

1. 使用 AJAX
2. 使用 FormData API

使用FormData来封装表单的数据是一种非常方便的方式，同时支持二进制文件上传，但同时缺点也就是他是二进制的，而不是字符串。即被收集的数据是二进制，而不是字符串。当然也可以通过对象本身的一些方法来获取或者说遍历。

那么博主今天要谈的是表单上传的一些Content-Type以及对于的编码方式:

一个 html <form> 可以用以下四种方式发送：

* 使用 POST 方法，并设置 enctype 属性为 application/x-www-form-urlencoded (默认)
* 使用 POST 方法，并设置 enctype 属性为 text/plain
* 使用 POST 方法，并设置 enctype 属性为 multipart/form-data
* 使用 GET 方法（这种情况下 enctype 属性会被忽略）


#### application/x-www-form-urlencoded (default):

表单默认的一种编码方式:

最终传输的body：eg

```
Content-Type: application/x-www-form-urlencoded

foo=the+first+line%0D%0Athe+second+line&foo=bar
```

%0D%0A是回车换行符被url编码时候的码:

```
encodeURIComponent("\r\n") "%0D%0A"
如果只有单单的换行符，就只会转化成 "%0A"

注意空格位置被填充了+， 而在request param中，参数被encode为了 %20 
```



其实就和url encode的方式是一样的产生的数据结果是类似的，只不过数据在body中。

#### text/plain:

文本形式的传递：保存了换行，同时item之间也添加了换行

```
Content-Type: text/plain

foo=bar
baz=The first line.
The second line.
```

#### multipart/form-data:

```
Content-Type: multipart/form-data; boundary=---------------------------314911788813839


-----------------------------314911788813839
Content-Disposition: form-data; name="foo"

bar
-----------------------------314911788813839
Content-Disposition: form-data; name="baz"

The first line.
The second line.

-----------------------------314911788813839--
```

注意看到是有一个boundary，其实不难看出，就是每个item的分界符，类似于我们url编码中的&。

#### 同时，如果在form上面使用get方法，参数就会直接带到Request URL上。

```
foo=the+first+line%0D%0Athe+second+line&bar=ioo&foo=bar
```

#### application/json

***使用 POST 方法，并设置 XMLHttpRequest 对象的 Content-Type 属性为 application/json***

虽然没有被官方推荐，在form表单上设置被认为无效，但在使用XMLHttpRequest发送时浏览器是识别的。

```
{"ss":"cc","ww":"the f line \r\n the s line","wd":"key worf"}
```

其实也是一种序列化的结果，还是字符串不是？

## Koa处理

博主在使用Koa的中间件在对这些请求进行处理时，也发现了一些问题。

我发现co-busboy这个中间件在处理诸如application/x-www-form-urlencoded和form-date的格式是特别在行，特别是二进制。

而co-body，也可以处理application/x-www-form-urlencoded，在处理application/json以及text/plain等方面，可以选择co-body。

其实从**co-busboy**的Example可以看出一二:

```
var parse = require('co-busboy')

app.use(function* (next) {
  // the body isn't multipart, so busboy can't parse it
  if (!this.request.is('multipart/*')) return yield next

  var parts = parse(this)
  var part
  while (part = yield parts) {
    if (part.length) {
      // arrays are busboy fields
      console.log('key: ' + part[0])
      console.log('value: ' + part[1])
    } else {
      // otherwise, it's a stream
      part.pipe(fs.createWriteStream('some file.txt'))
    }
  }
  console.log('and we are done parsing the form!')
})
```

parts -> boundary?

yield -> 流读取？ 

。。。当然，这只是作者的些许猜测。


当然，还有一个koa-bodyparser:

> A body parser for koa, base on co-body. support json, form and text type body.


## 后话

关于form content-type一点点初步的总结，就到这儿。但探索不会止步。其实挺看好这些中间件和NodeJS的，感觉其更干净，复杂却又高度灵活性，可定制。颇具安全感不是嘛？

好了，不瞎逼逼了。

窗外一片安宁祥和，这该是自然的声音，自然的味道。


## Refers

1. [Using XMLHttpRequest](https://developer.mozilla.org/zh-CN/docs/Web/API/XMLHttpRequest/Using_XMLHttpRequest)	
2. [FormData](https://developer.mozilla.org/en-US/docs/Web/API/FormData)


—— Tth End

—— LanceLou 2016.04.09
