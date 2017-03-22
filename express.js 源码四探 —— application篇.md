#### 本篇
本篇是这个系列的第四篇，谢谢各位的建议  

#### 环境
保持与之前的版本一致，源码版本是[express.js 4.14.1](https://github.com/expressjs/express/tree/4.14.1)

----
##application源码
###### app.init
[戳这里](https://github.com/expressjs/express/blob/4.14.1/lib/application.js#L37)，初始化app的一些参数 
```
//导出的是这个app实例  
var app = exports = module.exports = {};

//这个后面再讨论，关于代理  
var trustProxyDefaultSymbol = '@@symbol:trust_proxy_default';

//cache、engines、settings属性设置为空对象
app.init = function init() {
  this.cache = {};
  this.engines = {};
  this.settings = {};
  //运行下面的默认配置函数
  this.defaultConfiguration();
};
```

###### app.defaultConfiguration
**一**
```
// 如果没有设置NODE_ENE，默认设置成development，即开发环境
var env = process.env.NODE_ENV || 'development';
...
this.set('env', env);
```
**二**
```
// 在http头中添加x-powered-by，让开发人员知道webserver框架
this.enable('x-powered-by');
```
**三**  
etag是跟文件相关的一种标记，上篇讲到了etag的作用  
一种是weak精确到秒级的，可以避免用户同一秒中不停刷新每次都重新下载资源  
另一种strong的精确到毫秒  
```
this.set('etag', 'weak');
```
**四**  
主要看看
```
this.set('query parser', 'extended');
```
这个 extended 设置，会影响代码可以在这里看到[Line 368](https://github.com/expressjs/express/blob/4.14.1/lib/application.js#L368)
```
//app.set
switch (setting) {
    case 'etag':
      this.set('etag fn', compileETag(val));
      break;
    case 'query parser':
      this.set('query parser fn', compileQueryParser(val));
      break;
    case 'trust proxy':
      this.set('trust proxy fn', compileTrust(val));
    ...
    break;
}
```
可以看到**app.set**函数里面，对query parser是有特殊处理，配置了'query parser fn'    
具体配置的是什么，继续看看这个compileQueryParser把val='extended'做了哪些处理  
[./lib/utils.js Line197](https://github.com/expressjs/express/blob/4.14.1/lib/utils.js#L197)  
```
exports.compileQueryParser = function compileQueryParser(val) {
  ...
  switch (val) {
    ...
    case 'extended':
      fn = parseExtendedQueryString;
      break;
    ...
  }
  return fn;
}

```
上面的代码可知compileQueryParse返回了函数，所以'query parser fn'配置了一个函数  
这个函数具体实现是什么  
继续翻[./lib/utils.js Line284](https://github.com/expressjs/express/blob/4.14.1/lib/utils.js#L284)，代码如下
```
function parseExtendedQueryString(str) {
  return qs.parse(str, {
    allowPrototypes: true
  });
}
```
var qs = require('qs') 明白了知道这个qs不是nodejs自带库querystring  
那么这个allowPrototypes是啥，继续找到这个[qs库](https://github.com/ljharb/qs/tree/v6.2.0)
```
By default parameters that would overwrite properties on the object prototype are ignored, if you wish to keep the data from those fields either use plainObjects as mentioned above, or set allowPrototypes to true which will allow user input to overwrite those properties. WARNING It is generally a bad idea to enable this option as it can cause problems when attempting to use the properties that have been overwritten. Always be careful with this option.
```
这个是什么意思呢，试试querystring，和这个qs库就知道了
```
const qs = require('./lib/index.js');
const querystring = require('querystring');

console.log(qs.parse('a[w]=s'));
console.log(querystring.parse('a[w]=s'));

{ a: { w: 's' } }
{ 'a[w]': 's' }
```
就是说nodejs querystring默认的解析方式，在对象上写属性是被忽略的，但是qs可以做到，只要设置了allowPrototypes配置为true  

**五**
```
this.set('subdomain offset', 2);
```
这个是为了给[./request.js](https://github.com/expressjs/express/blob/4.14.1/lib/request.js#L382) req.subdomain使用的  
比如对于域名 `"tobi.ferrets.example.com"`  
如果设置为2，那么req.subdomains = `["ferrets", "tobi"]`  
如果设置为3，那么req.subdomains is `["tobi"]`

**六**  
不采用托管代理模式    
```
this.set('trust proxy', false);
```
trust proxy 这个属性设置false（默认值），那么从代理服务器转发过来的报文（携带原始报文的req信息），server会认为代理服务器就是初始发报文的client
(或者取值req.ip、protocol等的时候，实际取得是代理服务器的ip、protocol)   

主要原理是看http头  
X-Forwarded-For: client, proxy1, proxy2, proxy3，代表着所有的客户端和代理服务器地址  
X-Forwarded-Host: client，代表了最原始的报文中的客户端地址  
解析头的部分代码在这里 [forwarded#L30](https://github.com/jshttp/forwarded/blob/master/index.js#L30)  
比如，下面的代码就是请求报文中取ip
```
defineGetter(req, 'ip', function ip(){
  var trust = this.app.get('trust proxy fn');
  return proxyaddr(this, trust);
});
```

**七**   
挂上事件驱动，监听mount，具体作用见下文
```
  this.on('mount', function onmount(parent) {
    // inherit trust proxy
    if (this.settings[trustProxyDefaultSymbol] === true
      && typeof parent.settings['trust proxy fn'] === 'function') {
      delete this.settings['trust proxy'];
      delete this.settings['trust proxy fn'];
    }

    // inherit protos
    this.request.__proto__ = parent.request;
    this.response.__proto__ = parent.response;
    this.engines.__proto__ = parent.engines;
    this.settings.__proto__ = parent.settings;
  });
```

**八**   
```
  //把配置放一份在locals属性上
  this.locals.settings = this.settings;

  // default configuration
  this.set('view', View);
  this.set('views', resolve('views'));

  // 支持jsonp来实现跨域，安全问题，详见之前的第三篇
  this.set('jsonp callback name', 'callback');

  if (env === 'production') {
    this.enable('view cache');
  }

  //app.router这个方法在3以后的版本就不建议使用了
  Object.defineProperty(this, 'router', {
    get: function() {
      throw new Error('\'app.router\' is deprecated!\nPlease see the 3.x to 4.x migration guide for details on how to update your app.');
    }
  });
```

###### app.lazyrouter
```
if (!this._router) {
  this._router = new Router({
    caseSensitive: this.enabled('case sensitive routing'),
    strict: this.enabled('strict routing')
  });

  this._router.use(query(this.get('query parser fn')));
  this._router.use(middleware.init(this));
} 
```
懒加载函数，前面的第一章有提过  
所有的非路由中间件、路由中间件都被收集在_router上  
caseSensitive 设置为大小写敏感  
strict 设置为严格模式，什么是严格模式？主要指末尾的斜杠，如果路由中间件app.get('/user')，那么请求get('/user/')是匹配不上的  
添加两个express的默认中间件，query 和 init  
注意这里的query中间件，跟上面的[app.defaultConfiguration](#app.defaultConfiguration)第四点刚好对应上，自定义的qs  

###### app.handle
```
app.handle = function handle(req, res, callback) {
  var router = this._router;

  // final handler
  var done = callback || finalhandler(req, res, {
    env: this.get('env'),
    onerror: logerror.bind(this)
  });

  // no routes
  if (!router) {
    debug('no routes defined on app');
    done();
    return;
  }

  router.handle(req, res, done);
};
``` 
中间件触发处理函数，本质上是触发了router.handle的函数，注意一般的404也是从这个done来的

###### app.use
这个use这个函数前面已经提到很多次，也是开发人员经常用到的接口之一    
**一**.
下面的代码主要是为了兼容app.use("/sss", ()=>{}) 和 app.use( ()=> {})两种用法，offset指的是参数中偏移到函数数组的位置，然后用slice.call把函数数组分割出  
```
var offset = 0;
var path = '/';
// default path to '/'
// disambiguate app.use([fn])
if (typeof fn !== 'function') {
  var arg = fn;
  while (Array.isArray(arg) && arg.length !== 0) {
    arg = arg[0];
  }
  // first arg is the path
  if (typeof arg !== 'function') {
    offset = 1;
    path = fn;
  }
}
var fns = flatten(slice.call(arguments, offset));
```
**二**.  
懒加载，前面也提到过，主要是给app添加_router#Router这个属性，同时添加中间件express.Init和qs两个中间件
```
// setup router
this.lazyrouter();
var router = this._router;
```
**三**.  
这里就是把传入的fns参数数组，都封装成Layer对象，然后一个个压入__router属性中    
正常流程来讲，走到下面的//non-express app的地方，就可以return了  
```
fns.forEach(function (fn) {
  // non-express app
  if (!fn || !fn.handle || !fn.set) {
    return router.use(path, fn);
  }
  ...
}, this);
```
这里面复杂的地方在于下面的代码，关于mount的，引申的问题比如orig这个临时变量  
要弄清这个逻辑，先看个例子  
```
var express = require('./lib/express');
var app = express();
var admin = express();

admin.on('mount', function (parent) {
  console.log('Admin Mounted');
  console.log(parent); // refers to the parent app
});

admin.get('/ss', function (req, res) {
  res.send('send ss in admin');
});

app.use('/admin', admin);
app.use('/wwww', function (req, res) {
  res.send('send wwww in app');
});

app.use('/wwww2', function (req, res) {
  res.send('send wwww2 in app');
});


app.listen(3000); 
admin.listen(3001);
```
上面的代码逻辑，可以理解为有一个名叫admin的子模块(不知道具体叫什么，暂且叫做“子模块”)挂载在app的父模块上，但是app.use为什么能够用一个express()对象作为传入参数，并且成功挂载？  
实际上，express()返回的admin对象本质上就是一个function，可以看[express.js#L37](https://github.com/expressjs/express/blob/4.14.1/lib/express.js#L37)，它也是express框架处理req, res的入口，因此这里可以把报文传给子模块处理  
这个admin比其他function多拥有handle、set属性，所以在app.use里面逻辑会走到[下面的代码](https://github.com/expressjs/express/blob/4.14.1/lib/application.js#L216)(注意admin.use自己不会走到这，除非它也有子模块)
```
  fn.mountpath = path;
  fn.parent = this;

  // restore .app property on req and res
  router.use(path, function mounted_app(req, res, next) {
    var orig = req.app;
    fn.handle(req, res, function (err) {
      req.__proto__ = orig.request;
      res.__proto__ = orig.response;
      next(err);
    });
  });

  // mounted an app
  fn.emit('mount', this);
```
样例中，代码逻辑走到这里时，app对象里面的fn就是admin对象，fn发出的'mount'，可以被事例中的admin.on('mount', ()=>{})捕捉到  
当然好玩的是，admin本身又可以作为一个父模块监听另一个端口

###### app.route
返回一个route对象（注意区分router对象）
```
app.route = function route(path) {
  this.lazyrouter();
  return this._router.route(path);
};
```
这个函数有啥用呢，可以看这里[application.js#L471](https://github.com/expressjs/express/blob/4.14.1/lib/application.js#L471)  
```
methods.forEach(function(method){
  app[method] = function(path){
    if (method === 'get' && arguments.length === 1) {
      // app.get(setting)
      return this.set(path);
    }

    this.lazyrouter();

    //第一步
    var route = this._router.route(path);
    //第二步
    route[method].apply(route, slice.call(arguments, 1));
    return this;
  };
});
```
上面代码是创建各种路由中间件出来的地方  
第一步：先弄一个route对象（路由中间件）出来 ，把app[method]的函数参数（使用Layer对象包裹）push到这个路由中间件的stack数组里  
第二步：触发时候，就调用apply来执行传入的回调函数  

##### 其他
app.engine 传入一个html的渲染模板   
app.param  对与特定的参数，执行传入的函数   
app.set 设置一些属性，其中特别的etag/query parser/trust proxy上篇和上面都有讲到过  
app.path 因为挂载（上面的父子模块情况），所以path是需要拼接的  
app.enabled/disabled/enable/disable 检查某个设置是否已经“使能”  
app.all 匹配所有该路径下的请求  
app.render 渲染传参app.render('email', { name: 'Tobi' }, () => {})  



----
#### 参考  
[REST笔记(五):你应该知道的HTTP头------ETag](http://www.cnblogs.com/tyb1222/archive/2011/12/24/2300246.html)