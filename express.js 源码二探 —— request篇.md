#### 接上篇初探
上篇写的很乱，只是个人临时决定写的。主要试图从app.use和app.[method]之间区别着手，来解释express的中间件和路由的实现逻辑。  

#### 本篇
本篇很有可能依然保持上篇的发散性思维，所以各位读者可能要小心阅读，避免吐槽而亡。

#### 环境
保持与上篇一致，源码版本是[express.js 4.14.1](https://github.com/expressjs/express/tree/4.14.1)

----  
## request源码  
打开[request.js](https://github.com/expressjs/express/blob/4.14.1/lib/request.js)，从上往下看代码

##### IncomingMessage
```js
var req = exports = module.exports = {
  __proto__: http.IncomingMessage.prototype
};
```
[代码具体戳这里](https://github.com/expressjs/express/blob/4.14.1/lib/request.js#L31)   
我们要**导出request这个对象，跟http.IncomingMessage有什么关系，而且是挂在request的\_\_proto\_\_原型链上**  
我们知道nodejs提供的原生http对象通常是这么用的
```
var http = require('http');
http.createServer( (function(req, res) { 
    res.end("hello world");
}) ).listen("3000");
```  
那么我们看看nodejs的源代码里面的http.js，找到[下面的代码](https://github.com/nodejs/node/blob/v6.9.4/lib/http.js#L8)  
```
const server = require('_http_server');
...
const Server = exports.Server = server.Server;

exports.createServer = function(requestListener) {
  return new Server(requestListener);
};
```
上面可以看到是回调函数是从\_http\_server.js里面出来的，那么继续翻[\_http\_server.js](https://github.com/nodejs/node/blob/v6.9.4/lib/_http_server.js#L229)的代码，如下
```
function Server(requestListener) {
  ...
  if (requestListener) {
    this.addListener('request', requestListener);
  }
  ...
}
```
可以看到是观察者模式，那么可以\_http\_server.js里搜索关键字 "emit('request'"，迅速找到这里的触发函数
```

if (req.headers.expect !== undefined &&
        (req.httpVersionMajor === 1 && req.httpVersionMinor === 1)) {
    ...
    self.emit('request', req, res);
    ...
} else {
    ...
    self.emit('request', req, res);
    ...
}
```
可以看到这个req是从这里传递下去的，下面看到[380行](https://github.com/nodejs/node/blob/v6.9.4/lib/_http_server.js#L463)
```
function onParserExecuteCommon(ret, d) {
    ...
    var req = parser.incoming;
    ...
}
```
当然还有好多处可以看见var req = parser.incoming;那么去看看incoming怎么来的，去翻\_http\_commong.js [60行](https://github.com/nodejs/node/blob/v6.9.4/lib/_http_common.js#L60)
```
parser.incoming = new IncomingMessage(parser.socket);
```
那么可以知道，其实request就是构造器IncomingMessage new 出来的一个对象  
这里也就解释了我们express的req为什么要在原型链上加上**IncomingMessage**  


##### req.get && req.header
这一部分，就是返回http头的字段，比较有意思的就是referer  
那个规范http作者当年给拼错了referer，所以导致现在referrer与referer表达的是同一个意思    
```
  switch (lc) {
    case 'referer':
    case 'referrer':
      return this.headers.referrer
        || this.headers.referer;
    default:
      return this.headers[lc];
  }
```

##### req.accepts
这个是个accepts库，主要用于验证http的资源类型  
如果匹配返回true；相反则返回undefined，这时候response应该返回406  
还有其他的几个accepts，不一一介绍了  

##### req.range
下载的http头中可以携带range字段，指明想要下载资源的哪一部分
```
Range: bytes=500-999
```
如果资源总的大小是10000，而且正常返回http 200，那么res的头里面应该是这样的  
```
Content-Range: bytes 500-999/10000
```

那req.range这个函数本身是干什么的呢？
```
req.range = function range(size, options) {
  var range = this.get('Range');
  if (!range) return;
  return parseRange(size, range, options);
};
```
翻译一下注解及翻阅源码:  

* 如果req的http头中Range字段未给出，该函数返回undefined
* 如果这个资源不能划分成size范围内的，该函数返回-1
* 如果语法错误，该函数返回-2
* 正常的返回，如下例
```
Range: bytes=500-600,601-999 //http头Range字段

//那么该函数返回
[
    {"start" : 500, "end" : 600},
    {"start" : 601, "end" : 999}
]
//并且返回的数组对象有一个属性type = bytes, 这里值为bytes是因为请求头Range中里面的"bytes" 
```

##### req.param
这个比较简单， param(name, defaultValue)就是在req.params、req.body、req.query等对象里面取这个name的属性值，如果都找不到，那就返回defaultValue    
req.params、body、query这三个分别指解析了如下部分

 *  路由的占位: _/user/:id_
 *  body里面的数据: id=12, {"id":12}
 *  url里面的参数: ?id=12

##### req.is  
就是判断请求的http数据类型，直接拿函数注释例子就好
```
// With Content-Type: text/html; charset
req.is('html');
req.is('text/html');
req.is('text/*');
// => true


// When Content-Type is application/json
req.is('json');
req.is('application/json');
req.is('application/*');
// => true
req.is('html');
// => false
```

##### defineGetter
剩下的函数都是get访问器  

* protocol  //http或者https
* secure   //返回true或者false，安全就是https协议
* ip     //返回ip地址
* ips    //ip地址，包括中间代理的ip
* subdomains  //子域名数组，tobi.ferrets.example.com返回 ["ferrets", "tobi"]
* path  //url里面的路径
* hostname //主机名
* host  //上面一样，后面要被废弃
* fresh // 根据Last-Modified和ETag判断客户端是否是最新缓存，没有的话顺手缓存一下
* stale //同上，但是判断的是客户端缓存是否过期
* xhr //判断是否是xmlhttprequest（ajax）发送的请求

就是利用defineProperty添加get访问器，当尝试访问属性的时候会触发，由于这些不可写，所以没有set访问器

##### 结合
那么nodejs http.createServer里面的参数req对象，怎么就和现在我们定义的req对象结合上去了呢？    
看中间件/lib/middleware/init.js
```
exports.init = function(app){
  return function expressInit(req, res, next){
    ... 
    //通过原型链，赋给req对象的
    req.__proto__ = app.request;
    res.__proto__ = app.response;
    ...
  };
};
```
