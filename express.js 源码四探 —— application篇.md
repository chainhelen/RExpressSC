#### ��ƪ
��ƪ�����ϵ�еĵ���ƪ��лл��λ�Ľ���  

#### ����
������֮ǰ�İ汾һ�£�Դ��汾��[express.js 4.14.1](https://github.com/expressjs/express/tree/4.14.1)

----
##applicationԴ��
###### app.init
[������](https://github.com/expressjs/express/blob/4.14.1/lib/application.js#L37)����ʼ��app��һЩ���� 
```
//�����������appʵ��  
var app = exports = module.exports = {};

//������������ۣ����ڴ���  
var trustProxyDefaultSymbol = '@@symbol:trust_proxy_default';

//cache��engines��settings��������Ϊ�ն���
app.init = function init() {
  this.cache = {};
  this.engines = {};
  this.settings = {};
  //���������Ĭ�����ú���
  this.defaultConfiguration();
};
```

###### app.defaultConfiguration
**һ**
```
// ���û������NODE_ENE��Ĭ�����ó�development������������
var env = process.env.NODE_ENV || 'development';
...
this.set('env', env);
```
**��**
```
// ��httpͷ�����x-powered-by���ÿ�����Ա֪��webserver���
this.enable('x-powered-by');
```
**��**  
etag�Ǹ��ļ���ص�һ�ֱ�ǣ���ƪ������etag������  
һ����weak��ȷ���뼶�ģ����Ա����û�ͬһ���в�ͣˢ��ÿ�ζ�����������Դ  
��һ��strong�ľ�ȷ������  
```
this.set('etag', 'weak');
```
**��**  
��Ҫ����
```
this.set('query parser', 'extended');
```
��� extended ���ã���Ӱ�������������￴��[Line 368](https://github.com/expressjs/express/blob/4.14.1/lib/application.js#L368)
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
���Կ���**app.set**�������棬��query parser�������⴦��������'query parser fn'    
�������õ���ʲô�������������compileQueryParser��val='extended'������Щ����  
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
����Ĵ����֪compileQueryParse�����˺���������'query parser fn'������һ������  
�����������ʵ����ʲô  
������[./lib/utils.js Line284](https://github.com/expressjs/express/blob/4.14.1/lib/utils.js#L284)����������
```
function parseExtendedQueryString(str) {
  return qs.parse(str, {
    allowPrototypes: true
  });
}
```
var qs = require('qs') ������֪�����qs����nodejs�Դ���querystring  
��ô���allowPrototypes��ɶ�������ҵ����[qs��](https://github.com/ljharb/qs/tree/v6.2.0)
```
By default parameters that would overwrite properties on the object prototype are ignored, if you wish to keep the data from those fields either use plainObjects as mentioned above, or set allowPrototypes to true which will allow user input to overwrite those properties. WARNING It is generally a bad idea to enable this option as it can cause problems when attempting to use the properties that have been overwritten. Always be careful with this option.
```
�����ʲô��˼�أ�����querystring�������qs���֪����
```
const qs = require('./lib/index.js');
const querystring = require('querystring');

console.log(qs.parse('a[w]=s'));
console.log(querystring.parse('a[w]=s'));

{ a: { w: 's' } }
{ 'a[w]': 's' }
```
����˵nodejs querystringĬ�ϵĽ�����ʽ���ڶ�����д�����Ǳ����Եģ�����qs����������ֻҪ������allowPrototypes����Ϊtrue  

**��**
```
this.set('subdomain offset', 2);
```
�����Ϊ�˸�[./request.js](https://github.com/expressjs/express/blob/4.14.1/lib/request.js#L382) req.subdomainʹ�õ�  
����������� `"tobi.ferrets.example.com"`  
�������Ϊ2����ôreq.subdomains = `["ferrets", "tobi"]`  
�������Ϊ3����ôreq.subdomains is `["tobi"]`

**��**  
�������йܴ���ģʽ    
```
this.set('trust proxy', false);
```
trust proxy �����������false��Ĭ��ֵ������ô�Ӵ��������ת�������ı��ģ�Я��ԭʼ���ĵ�req��Ϣ����server����Ϊ������������ǳ�ʼ�����ĵ�client
(����ȡֵreq.ip��protocol�ȵ�ʱ��ʵ��ȡ���Ǵ����������ip��protocol)   

��Ҫԭ���ǿ�httpͷ  
X-Forwarded-For: client, proxy1, proxy2, proxy3�����������еĿͻ��˺ʹ����������ַ  
X-Forwarded-Host: client����������ԭʼ�ı����еĿͻ��˵�ַ  
����ͷ�Ĳ��ִ��������� [forwarded#L30](https://github.com/jshttp/forwarded/blob/master/index.js#L30)  
���磬����Ĵ��������������ȡip
```
defineGetter(req, 'ip', function ip(){
  var trust = this.app.get('trust proxy fn');
  return proxyaddr(this, trust);
});
```

**��**   
�����¼�����������mount���������ü�����
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

**��**   
```
  //�����÷�һ����locals������
  this.locals.settings = this.settings;

  // default configuration
  this.set('view', View);
  this.set('views', resolve('views'));

  // ֧��jsonp��ʵ�ֿ��򣬰�ȫ���⣬���֮ǰ�ĵ���ƪ
  this.set('jsonp callback name', 'callback');

  if (env === 'production') {
    this.enable('view cache');
  }

  //app.router���������3�Ժ�İ汾�Ͳ�����ʹ����
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
�����غ�����ǰ��ĵ�һ�������  
���еķ�·���м����·���м�������ռ���_router��  
caseSensitive ����Ϊ��Сд����  
strict ����Ϊ�ϸ�ģʽ��ʲô���ϸ�ģʽ����Ҫָĩβ��б�ܣ����·���м��app.get('/user')����ô����get('/user/')��ƥ�䲻�ϵ�  
�������express��Ĭ���м����query �� init  
ע�������query�м�����������[app.defaultConfiguration](#app.defaultConfiguration)���ĵ�պö�Ӧ�ϣ��Զ����qs  

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
�м���������������������Ǵ�����router.handle�ĺ�����ע��һ���404Ҳ�Ǵ����done����

###### app.use
���use�������ǰ���Ѿ��ᵽ�ܶ�Σ�Ҳ�ǿ�����Ա�����õ��Ľӿ�֮һ    
**һ**.
����Ĵ�����Ҫ��Ϊ�˼���app.use("/sss", ()=>{}) �� app.use( ()=> {})�����÷���offsetָ���ǲ�����ƫ�Ƶ����������λ�ã�Ȼ����slice.call�Ѻ�������ָ��  
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
**��**.  
�����أ�ǰ��Ҳ�ᵽ������Ҫ�Ǹ�app���_router#Router������ԣ�ͬʱ����м��express.Init��qs�����м��
```
// setup router
this.lazyrouter();
var router = this._router;
```
**��**.  
������ǰѴ����fns�������飬����װ��Layer����Ȼ��һ����ѹ��__router������    
���������������ߵ������//non-express app�ĵط����Ϳ���return��  
```
fns.forEach(function (fn) {
  // non-express app
  if (!fn || !fn.handle || !fn.set) {
    return router.use(path, fn);
  }
  ...
}, this);
```
�����渴�ӵĵط���������Ĵ��룬����mount�ģ�������������orig�����ʱ����  
ҪŪ������߼����ȿ�������  
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
����Ĵ����߼����������Ϊ��һ������admin����ģ��(��֪�������ʲô�����ҽ�������ģ�顱)������app�ĸ�ģ���ϣ�����app.useΪʲô�ܹ���һ��express()������Ϊ������������ҳɹ����أ�  
ʵ���ϣ�express()���ص�admin�������Ͼ���һ��function�����Կ�[express.js#L37](https://github.com/expressjs/express/blob/4.14.1/lib/express.js#L37)����Ҳ��express��ܴ���req, res����ڣ����������԰ѱ��Ĵ�����ģ�鴦��  
���admin������function��ӵ��handle��set���ԣ�������app.use�����߼����ߵ�[����Ĵ���](https://github.com/expressjs/express/blob/4.14.1/lib/application.js#L216)(ע��admin.use�Լ������ߵ��⣬������Ҳ����ģ��)
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
�����У������߼��ߵ�����ʱ��app���������fn����admin����fn������'mount'�����Ա������е�admin.on('mount', ()=>{})��׽��  
��Ȼ������ǣ�admin�����ֿ�����Ϊһ����ģ�������һ���˿�

###### app.route
����һ��route����ע������router����
```
app.route = function route(path) {
  this.lazyrouter();
  return this._router.route(path);
};
```
���������ɶ���أ����Կ�����[application.js#L471](https://github.com/expressjs/express/blob/4.14.1/lib/application.js#L471)  
```
methods.forEach(function(method){
  app[method] = function(path){
    if (method === 'get' && arguments.length === 1) {
      // app.get(setting)
      return this.set(path);
    }

    this.lazyrouter();

    //��һ��
    var route = this._router.route(path);
    //�ڶ���
    route[method].apply(route, slice.call(arguments, 1));
    return this;
  };
});
```
��������Ǵ�������·���м�������ĵط�  
��һ������Ūһ��route����·���м�������� ����app[method]�ĺ���������ʹ��Layer���������push�����·���м����stack������  
�ڶ���������ʱ�򣬾͵���apply��ִ�д���Ļص�����  

##### ����
app.engine ����һ��html����Ⱦģ��   
app.param  �����ض��Ĳ�����ִ�д���ĺ���   
app.set ����һЩ���ԣ������ر��etag/query parser/trust proxy��ƪ�����涼�н�����  
app.path ��Ϊ���أ�����ĸ���ģ�������������path����Ҫƴ�ӵ�  
app.enabled/disabled/enable/disable ���ĳ�������Ƿ��Ѿ���ʹ�ܡ�  
app.all ƥ�����и�·���µ�����  
app.render ��Ⱦ����app.render('email', { name: 'Tobi' }, () => {})  



----
#### �ο�  
[REST�ʼ�(��):��Ӧ��֪����HTTPͷ------ETag](http://www.cnblogs.com/tyb1222/archive/2011/12/24/2300246.html)