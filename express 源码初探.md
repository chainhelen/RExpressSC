#### 问题
其实也没啥问题，就是一直用express.js，但是之前没有读过[源码](https://github.com/expressjs/express)，抽点时间看看

#### 环境
express 版本 是 [4.14.1](https://github.com/expressjs/express/releases/tag/4.14.1)
nodejs 版本是 v6.9.4
windows10 
vscode1.9.1 + nodejs调试插件
launch.json 配置如下
```
{
    "version": "0.2.0",
    "configurations": [
        {
            "type": "node",
            "request": "launch",
            "name": "启动程序",
            "program": "${workspaceRoot}/test.js",
            "cwd": "${workspaceRoot}"
            // 如果想看express代码里面的debug信息，设置下面的环境变量即可
            // "env" : {"DEBUG" : "*,-not_this"}
        },
        {
            "type": "node",
            "request": "attach",
            "name": "附加到进程",
            "port": 5858
        }
    ]
}
```

#### app.use
在express的主代码目录下，写一个test.js  
```
"use strict"
var express = require('./lib/express.js');
var app = express();

app.use('/main', (req, res, next) => {
    res.end("hello world\n");
    console.log("process /main");
});

app.listen(3000, () => {
    console.log("listening.... ")
});
``` 
我们在app.listen 那一行下一个断点  

![testjs1.png](http://imgqi.hacking.pub/express.js%20%E6%BA%90%E7%A0%81%E5%88%9D%E6%8E%A2/testjs1.png)  
可以看见左边app._router的stack数组里面有三个Layer，其中stack[2]就是我们的app.user("/main")产生的  
那么stack[0] 也就是那个handle是query中间件 是哪里来的呢？  
查了一下代码，是在application.js里面的
```
app.lazyrouter = function lazyrouter() {
  if (!this._router) {
    this._router = new Router({
      caseSensitive: this.enabled('case sensitive routing'),
      strict: this.enabled('strict routing')
    });

    this._router.use(query(this.get('query parser fn')));
    this._router.use(middleware.init(this));
  }
};
```
从这里也可以看见，express.js还保留了两个默认的中间件query和expressInit  

同时我们也可以得到一个结论就是，**express中的中间件 function 都是会被一个Layer对象封装，通过app._router.stack数组 push 该Layer对象**

#### app[method]
我们拿app.get来做测试，将test.js源代码修改如下
```
"use strict"
var express = require('./lib/express.js');
var app = express();

//app.use('/main', (req, res, next) => {
//    res.end("hello world\n");
//    console.log("/main");
//});

app.get('/main', (req, res, next) => {
    res.end("hello world\n");
    console.log("process main");
});

app.listen(3000, () => {
    console.log("listening.... ")
})
```
依旧在app.listen处下断点，着重比较一下app.use和app.get的区别  

首先看一下app.use的，下图就是 **app.use(query)** 产生的Layer对象，并保存在app._router.stack[0]位置  
特别注意一下，**Layer.handle就是我们传入的回调函数，并且route是undefined**   
 
![testjs2.png](http://imgqi.hacking.pub/express.js%20%E6%BA%90%E7%A0%81%E5%88%9D%E6%8E%A2/testjs2.png)  

再来看一下app.get的，是 **app.get('/main', function(){})** 产生的Layer对象，并保存在app.\_router.stack[2]的位置  
仔细看，**Layer.handle代码是"native code"，并且它的route属性并不是undefined，是一个Route对象**  
这个route跟app.\_router都有个stack属性，它里面也是保存着Layer对象数组 **=>** 从结果来看，app._router.stack[2].route.stack[0]是一个Layer对象，而这个Layer包裹着我们在app.get中传入回调函数  

![testjs3.png](http://imgqi.hacking.pub/express.js%20%E6%BA%90%E7%A0%81%E5%88%9D%E6%8E%A2/testjs3.png)

#### 初步总结  
**app.use**  
传入的参数（路径、回调函数）会被封装成Layer对象（其中route属性为undefined），push到app._router.stack  

**app.get**  
1.首先根据传入的路径封装一个Route对象，再对传入的回调函数封装成Layer对象，接着把这个Layer对象push到Route.stack里面去
2.再创建一个默认Layer(跟app.use里面Layer的同级)，把步骤一中的Route挂到这个Layer.route属性上面，而这个Layer对象，会被push到app._router.stack里面

举一下例子：  
```
"use strict"
var express = require('./lib/express.js');
var app = express();

app.use('/main', (req, res, next) => {
    res.end("hello world\n");
    console.log("/main");
});

app.get('/main', (req, res, next) => {
    res.end("hello world\n");
    console.log("process main");
});

app.listen(3000, () => {
    console.log("listening.... ")
})
```
生成的对象结构如下  

![testjs4.png](http://imgqi.hacking.pub/express.js%20%E6%BA%90%E7%A0%81%E5%88%9D%E6%8E%A2/testjs4.png)

#### app.use代码  
**1**.首先在test.js里面，app.use调用，那么来看app对象是怎么来的
```
var express = require('./lib/express.js');
var app = express();
```

**2**.上面可以看出是在./lib/expresss.js里面，而在./lib/express.js里面并没有直接定义use函数  
但是注意到下面的代码，它把proto的所有属性都导到app的属性上    
而proto是从./lib/application里面来的  
```
var proto = require('./application');
....
function createApplication() {
  ...
  mixin(app, proto, false);
  ...
}
```

**3**.继续查看./lib/application.js，就翻到了  
```
app.use = function use(fn) {
  ...
  // setup router
  this.lazyrouter();
  var router = this._router;

  fns.forEach(function (fn) {
    ...
    // restore .app property on req and res
    router.use(path, function mounted_app(req, res, next) {
      var orig = req.app;
      fn.handle(req, res, function (err) {
        req.__proto__ = orig.request;
        res.__proto__ = orig.response;
        next(err);
      });
    });
    ...
    // mounted an app
    fn.emit('mount', this);
  }, this);

  return this;
};
``` 
这里面使用了router.use，通过之前的分析，我们知道所有中间件（路由）都是直接或间接存储在app._router，那么接下来就是找router相关的代码  
顺便提一句，this.lazyrouter()就是加入query、expressinit中间件的入口 

**4**.继续翻./lib/router/index.js
```
proto.use = function use(fn) {
  ...
  for (var i = 0; i < callbacks.length; i++) {
    var fn = callbacks[i];
    ...
    var layer = new Layer(path, {
      sensitive: this.caseSensitive,
      strict: false,
      end: false
    }, fn);

    layer.route = undefined;

    this.stack.push(layer);
  }
  ...
  return this;
};
```
这一段代码就能很清晰看到，是利用Layer对象封装了传入的回调函数，然后app.stack push了这个Layer实例对象，并且这个Layer.route=undefined  

#### app[method]  代码

**1**. 拿app.get来看，在./lib/application.js中能找到这样的代码  
```
methods.forEach(function(method){
  app[method] = function(path){
    ...
    this.lazyrouter();

    var route = this._router.route(path);
    route[method].apply(route, slice.call(arguments, 1));
    return this;
    ...
  };
});
```
methods是一组http请求的类型，包括GET、PUT、POST等等，于是app.get的属性就被定义了  
其实这里也可以看到懒加载this.lazyrouter()，如果testjs代码里先调用app.get后调用app.use，则query、expressinit两个中间件是这里触发加入的  
根据 var route = this._router.route(path) 这一行代码，可以知道需要翻./lib/router/index.js的源码    

**2**. 去./lib/router/index.js 看_router.route的函数实现
```
proto.route = function route(path) {
  var route = new Route(path);

  var layer = new Layer(path, {
    sensitive: this.caseSensitive,
    strict: this.strict,
    end: true
  }, route.dispatch.bind(route));

  layer.route = route;

  this.stack.push(layer);
  return route;
};  
```
这里看到route对象的创建，layer.route = route，而且this._route.stack push了这个layer  
那么根据之前现象可以**猜想** var route = new Route(path)，也是会有一个stack属性，并且这个stack里面都是Layer对象 

**3**.接着去翻./lib/router/route.js 
```
function Route(path) {
  this.path = path;
  this.stack = [];

  debug('new %s ------- Route', path);

  // route handlers for various http methods
  this.methods = {};
}
```
这个构造函数里面只有stack，但是是个空数组，跟猜想的不太一样，继续在改文件里面找  

```
methods.forEach(function(method){
  Route.prototype[method] = function(){
    var handles = flatten(slice.call(arguments));

    for (var i = 0; i < handles.length; i++) {
    ...

      var layer = Layer('/', {}, handle);
      layer.method = method;

      this.methods[method] = true;
      this.stack.push(layer);
    }
    ...
    return this;
  };
});
```
这段代码运行，会定义Route.get，而且可以看到route.stack 会 push(layer)，但是本身Route.get这个函数是什么时候运行的呢？毕竟这里只是定义了Route.get

**4**. 还得看/lib/application.js  
上面 第**1**部分 可以看到这样的代码
```
route[method].apply(route, slice.call(arguments, 1));
```
是的，就是这里执行的  

----
**未经同意，禁止转载**  
**by chainhelen**  
