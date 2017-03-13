#### ����
��ʵҲûɶ���⣬����һֱ��express.js������֮ǰû�ж���[Դ��](https://github.com/expressjs/express)�����ʱ�俴��

#### ����
express �汾 �� [4.14.1](https://github.com/expressjs/express/releases/tag/4.14.1)
nodejs �汾�� v6.9.4
windows10 
vscode1.9.1 + nodejs���Բ��
launch.json ��������
```
{
    "version": "0.2.0",
    "configurations": [
        {
            "type": "node",
            "request": "launch",
            "name": "��������",
            "program": "${workspaceRoot}/test.js",
            "cwd": "${workspaceRoot}"
            // ����뿴express���������debug��Ϣ����������Ļ�����������
            // "env" : {"DEBUG" : "*,-not_this"}
        },
        {
            "type": "node",
            "request": "attach",
            "name": "���ӵ�����",
            "port": 5858
        }
    ]
}
```

#### app.use
��express��������Ŀ¼�£�дһ��test.js  
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
������app.listen ��һ����һ���ϵ�  

![testjs1.png](http://imgqi.hacking.pub/express.js%20%E6%BA%90%E7%A0%81%E5%88%9D%E6%8E%A2/testjs1.png)  
���Կ������app._router��stack��������������Layer������stack[2]�������ǵ�app.user("/main")������  
��ôstack[0] Ҳ�����Ǹ�handle��query�м�� �����������أ�  
����һ�´��룬����application.js�����
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
������Ҳ���Կ�����express.js������������Ĭ�ϵ��м��query��expressInit  

ͬʱ����Ҳ���Եõ�һ�����۾��ǣ�**express�е��м�� function ���ǻᱻһ��Layer�����װ��ͨ��app._router.stack���� push ��Layer����**

#### app[method]
������app.get�������ԣ���test.jsԴ�����޸�����
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
������app.listen���¶ϵ㣬���رȽ�һ��app.use��app.get������  

���ȿ�һ��app.use�ģ���ͼ���� **app.use(query)** ������Layer���󣬲�������app._router.stack[0]λ��  
�ر�ע��һ�£�**Layer.handle�������Ǵ���Ļص�����������route��undefined**   
 
![testjs2.png](http://imgqi.hacking.pub/express.js%20%E6%BA%90%E7%A0%81%E5%88%9D%E6%8E%A2/testjs2.png)  

������һ��app.get�ģ��� **app.get('/main', function(){})** ������Layer���󣬲�������app.\_router.stack[2]��λ��  
��ϸ����**Layer.handle������"native code"����������route���Բ�����undefined����һ��Route����**  
���route��app.\_router���и�stack���ԣ�������Ҳ�Ǳ�����Layer�������� **=>** �ӽ��������app._router.stack[2].route.stack[0]��һ��Layer���󣬶����Layer������������app.get�д���ص�����  

![testjs3.png](http://imgqi.hacking.pub/express.js%20%E6%BA%90%E7%A0%81%E5%88%9D%E6%8E%A2/testjs3.png)

#### �����ܽ�  
**app.use**  
����Ĳ�����·�����ص��������ᱻ��װ��Layer��������route����Ϊundefined����push��app._router.stack  

**app.get**  
1.���ȸ��ݴ����·����װһ��Route�����ٶԴ���Ļص�������װ��Layer���󣬽��Ű����Layer����push��Route.stack����ȥ
2.�ٴ���һ��Ĭ��Layer(��app.use����Layer��ͬ��)���Ѳ���һ�е�Route�ҵ����Layer.route�������棬�����Layer���󣬻ᱻpush��app._router.stack����

��һ�����ӣ�  
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
���ɵĶ���ṹ����  

![testjs4.png](http://imgqi.hacking.pub/express.js%20%E6%BA%90%E7%A0%81%E5%88%9D%E6%8E%A2/testjs4.png)

#### app.use����  
**1**.������test.js���棬app.use���ã���ô����app��������ô����
```
var express = require('./lib/express.js');
var app = express();
```

**2**.������Կ�������./lib/expresss.js���棬����./lib/express.js���沢û��ֱ�Ӷ���use����  
����ע�⵽����Ĵ��룬����proto���������Զ�����app��������    
��proto�Ǵ�./lib/application��������  
```
var proto = require('./application');
....
function createApplication() {
  ...
  mixin(app, proto, false);
  ...
}
```

**3**.�����鿴./lib/application.js���ͷ�����  
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
������ʹ����router.use��ͨ��֮ǰ�ķ���������֪�������м����·�ɣ�����ֱ�ӻ��Ӵ洢��app._router����ô������������router��صĴ���  
˳����һ�䣬this.lazyrouter()���Ǽ���query��expressinit�м������� 

**4**.������./lib/router/index.js
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
��һ�δ�����ܺ�����������������Layer�����װ�˴���Ļص�������Ȼ��app.stack push�����Layerʵ�����󣬲������Layer.route=undefined  

#### app[method]  ����

**1**. ��app.get��������./lib/application.js�����ҵ������Ĵ���  
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
methods��һ��http��������ͣ�����GET��PUT��POST�ȵȣ�����app.get�����Ծͱ�������  
��ʵ����Ҳ���Կ���������this.lazyrouter()�����testjs�������ȵ���app.get�����app.use����query��expressinit�����м�������ﴥ�������  
���� var route = this._router.route(path) ��һ�д��룬����֪����Ҫ��./lib/router/index.js��Դ��    

**2**. ȥ./lib/router/index.js ��_router.route�ĺ���ʵ��
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
���￴��route����Ĵ�����layer.route = route������this._route.stack push�����layer  
��ô����֮ǰ�������**����** var route = new Route(path)��Ҳ�ǻ���һ��stack���ԣ��������stack���涼��Layer���� 

**3**.����ȥ��./lib/router/route.js 
```
function Route(path) {
  this.path = path;
  this.stack = [];

  debug('new %s ------- Route', path);

  // route handlers for various http methods
  this.methods = {};
}
```
������캯������ֻ��stack�������Ǹ������飬������Ĳ�̫һ���������ڸ��ļ�������  

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
��δ������У��ᶨ��Route.get�����ҿ��Կ���route.stack �� push(layer)�����Ǳ���Route.get���������ʲôʱ�����е��أ��Ͼ�����ֻ�Ƕ�����Route.get

**4**. ���ÿ�/lib/application.js  
���� ��**1**���� ���Կ��������Ĵ���
```
route[method].apply(route, slice.call(arguments, 1));
```
�ǵģ���������ִ�е�  

----
**δ��ͬ�⣬��ֹת��**  
**by chainhelen**  
