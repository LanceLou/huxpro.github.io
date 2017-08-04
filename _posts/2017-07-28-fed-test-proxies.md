---
layout:     post
title:      "工程化之联调代理"
subtitle:   "NodeJS 工程化 前后端"
date:       2017-07-27 00:00
author:     "LanceLou"
header-img: "img/post-fed-Test-proxies.jpg"
catalog: true
tags:
    - JavaScript
    - 工程化
---

> 木有

## 初衷

工程化，是一个讲不完的问题，玩的溜，事半功倍，玩的不够六，事倍功半。其实我对工程化的理解，也仅限于皮毛，毕竟经验不多，比较菜。

mock server，是前端工程化较为重要的一环。对开发效率也有不错的提升！实现，也颇为简单，同时，基于本地server的前端开发人员测试服务(我把它称之为本地代码，远程API)也是一把开发利器，特别对于测试机，线上bug相关的前后端API测试，是一个不错的解决方案。

在我现在就职的某厂这边，有使用专门的cli来进行相关的一系列自动化解决方案，这里不做过多累赘，这里我主要以我对这方面的了解，以及我的gulp-forward，来进行进一步的探讨与介绍！

## mock server

相对来说，mock server的逻辑比较简单。也就是直接把前端的API请求进行处理，使用本地的mock数据进行response！

在我这边(gulp-forward)的mock主要是读取本地的.json文件，而对应的json文件是存放在与请求逻辑格式相关的文件目录下。

比如说我有一个请求url是: xhr/user/detail

那么在我们的mock目录下，就会有一个 xhr/user/detail.json文件，具体细节还需读者自行脑补哈！

在gulp-forward中，配合gulp-connect，我的server张这样:

```JavaScript
const mockProxy = (config ,req, res, next) => {
  let url = req.url;
  if (config.mockReg.test(url)) {
    const curPath = `${config.mockDir}${url}`;
    let body = '';
    let status = 404;
    fs.exists(curPath, (exists) => {
      if (exists) {
        body = fs.readFileSync(curPath, 'utf8');
        status = 200;
      } else {
        body = 'api handler no find';
      }
      res.writeHead(status, {
        'Content-Length': Buffer.byteLength(body),
        'Content-Type': 'application/json; charset=utf-8',
      });
      res.end(body);
    });
  } else {
    // pass it
    next();
  }
};
```

逻辑也很简单，根据请求对象的url字段，使用对应的正则，如果匹配，读取mock目录下的对应API json文件，不匹配，我们进行放行(调用next(), 甩锅给下一个middleware)。同时路径匹配，但文件不存在，-> 404。

当然，这种mock模式也存在诸多的不足，接口调用参数，方式没有进行校验。mock数据如何编写，或者说自动生成也是个问题。同时，自动化的流程，高度依赖于规范化，这是关键。

## remote API forward

本地代码，远程API转发。在测试机，线上出现bug时其用武之地就突显出来了。当然，双刃剑原则，没有完美的解决方案，同样也存在问题，这在后文会有提到！

转发，原则上来说我们是希望将API转发到远程server。但其实呢，你会发现一些问题。如果仅仅是API转发，token如何设定，登录呢，权限呢！从而会导致一系列的后端服务通过非API做的事情(set cookie, 权限等)都会被 "拦截"掉。

这时候解决方案其实也比较简单，或者说比较hack，你还可以说不计性能(realy?)!

取反，API代理到remote，其实我们可以换种思考，反过来是前端文件(JavaScript，css，script)拦截到本地，大概方式是，调用方给出拦截规则，注意这个规则是拦截本地的前端文件，其它一并转发到远程。上码。

```
const remoteProxy = (config, req, res, next) => {
  proxy.on('error', (err, req, res) => {
    try {
      res.writeHead(500, {
        'Content-Type': 'text/plain'
      });
      log(chalk.red(`${logPreStr} Error url: ${req.headers.host}/${req.url}`));
      res.end(err);
    } catch (e) {
      log(chalk.red(`${logPreStr} Error ${e}`));
    }
  });

  let url = req.url;
  let remoteUrl = `${config.remoteUrl}:${config.remotePort}`;
  const rule = getMatchedRule(config.rules, url);
  if (rule) { // matched: request to local
    var targetUrl = url.replace(rule.reg, rule.replace);
    req.url = targetUrl;
    next();
  } else { // unmatched: forward to remote
    remoteUrl += url;
    printLog(`${req.headers.host}/${req.url}`, remoteUrl);
    proxy.web(req, res, {
      target: remoteUrl,
      prependPath: false,
    }, (err) => {
      log(chalk.red(`${logPreStr} Error ${err}`));
    });
  }
};
```

这样，服务端的输出与设置其实是可以一并作用到本地浏览器的。作为服务端代理转发，跨域不会存在。但如果远程是https，我们还需要进行本地server的一些数字证书配置等。这些http-proxy都有提供。后续gulp-forward的更新也会增加相关配置项。同时，我们需要对gulp服务相关自动刷新(liveload或者说browsersync)的请求进行拦截至本地，以免被转发到远程。

## 总结

**震惊**, 这不是gulp-forward的软文，gulp-forward我也在使用并进行不断的优化，纵然，缺点诸多，但其实在工程化方面的每一点努力，也许不明显，却是一个细水长流的过程。

lancelou.com! 期待与你的更多交流。

-- 2017.08.04
