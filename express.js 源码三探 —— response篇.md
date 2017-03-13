#### ��ƪ
��ƪ�����ϵ�еĵ���ƪ��лл��λ�Ľ���  

#### ����
������֮ǰ�İ汾һ�£�Դ��汾��[express.js 4.14.1](https://github.com/expressjs/express/tree/4.14.1)

----
## responseԴ��  
###### Object.create
��[response.js](https://github.com/expressjs/express/blob/4.14.1/lib/response.js#L41)���������¿����룬[������������](https://github.com/expressjs/express/blob/4.14.1/lib/request.js#L41)  
```
var res = module.exports = {
  __proto__: http.ServerResponse.prototype
};
```
��һƪ���ǽ���request�����http.IncomingMessage�Ĺ�ϵ����ô���Ƶ�response��ServerResponseҲ��һ����  
����ע�����µ�express�������Ѿ��������������Object.create()��[�����µĴ���](https://github.com/expressjs/express/blob/5.0.0-alpha.5/lib/response.js#L41)
```
var res = Object.create(http.ServerResponse.prototype)
```
Object.create��E5�������һ���µĶ��󴴽���ʽ��������Ҳ�ǰ�res.\_\_proto\_\_ָ��ServerResponse.prototype

##### res.status
```
res.status = function status(code) {
  this.statusCode = code;
  return this;
};
```
����Ƚϼ򵥣���������http ����ͷ��״̬��

##### res.links
���������http ͷ��links�ֶ�
```
res.links = function(links){
  var link = this.get('Link') || '';
  if (link) link += ', ';
  return this.set('Link', link + Object.keys(links).map(function(rel){
    return '<' + links[rel] + '>; rel="' + rel + '"';
  }).join(', '));
};
```
����Ϊʲô����link������httpͷ�ֶΣ���֮ǰû����������������[ RFC 5988](http://www.ietf.org/rfc/rfc5988.txt)��link�ֶ���Ҫ�����Դ�Ĺ�ϵ

##### res.send 
```
  switch (typeof chunk) {
    // string defaulting to html
    case 'string':
      if (!this.get('Content-Type')) {
        this.type('html');
      }
      break;
    case 'boolean':
    case 'number':
    case 'object':
      if (chunk === null) {
        chunk = '';
      } else if (Buffer.isBuffer(chunk)) {
        if (!this.get('Content-Type')) {
          this.type('bin');
        }
      } else {
        return this.json(chunk);
      }
      break;
  }
```
res.sendͨ�������Ĳ���������Buffer��json��ʽ��string  

 * string���ͣ����û�ж���Content-Type�������÷�������Ϊhtml
 * Buffer���Ͱ����Ͷ���Ϊbin(������)
 * json��ʽ����ȥ����app.json����

����������Ĵ������¼���etag��ֵ 
```
  // populate ETag
  var etag;
  var generateETag = len !== undefined && app.get('etag fn');
  if (typeof generateETag === 'function' && !this.get('ETag')) {
    if ((etag = generateETag(chunk, encoding))) {
      this.set('ETag', etag);
    }
  }
```
Ȼ����ǲ鿴����
```

  // freshness
  if (req.fresh) this.statusCode = 304;
```
[��ȡreq.freshֵ��get������](https://github.com/expressjs/express/blob/4.14.1/lib/request.js#L448)
```
defineGetter(req, 'fresh', function(){
  var method = this.method;
  var s = this.res.statusCode;

  // GET or HEAD for weak freshness validation only
  if ('GET' !== method && 'HEAD' !== method) return false;

  // 2xx or 304 as per rfc2616 14.26
  if ((s >= 200 && s < 300) || 304 === s) {
    return fresh(this.headers, (this.res._headers || {}));
  }

  return false;
});
```
���Կ�����fresh�Ĵ��룬��Ҫ���ж�����  

  * ����ͷ��["if-none-match"]�ܲ���ƥ��*������res������["etag"]
  * res������['last-modified']�Ƿ�С�ڵ�������ͷ["if-modified-since"]

��ƥ���ϣ�fresh����Ϊtrue��Ҳ����head.status = 304

##### res.json
��Ҫ�Ƿ���json��ʽ�����ݣ�����json�������л���string��ʽ�󣬺����Ľ�β���ǵ��õ�res.send�����ַ���
```
  // settings
  var app = this.app;
  var replacer = app.get('json replacer');
  var spaces = app.get('json spaces');
  var body = stringify(val, replacer, spaces);
```

��������ʾ��val��key-value��js����stringify�����ϵ�����JSON.stringify�ĺ���  
**1** ����replacer�Ǻ������Ƕ�val����Ķ�Ӧkey�任val��  
�ȷ�˵
```
var val = { name: 'tobi', _id: 12345 };
var str = JSON.stringify(val, (key, value) => {
    return '_' == key[0]
        �� undefined
        :  val;
});
console.log(str);
//���Ϊ�ַ����� {name:'tobi'} 
```
**2** ����spaces�Ƕ�����ַ��������һЩ�������ո񡢻��С�
```
var str = JSON.stringify({name : "w"}, undefined, 2);
console.log(str);
//���
{
   name : "w"
}
```

##### res.jsonp
����jsonp��ʽ�����ݣ�ʲô��jsonp  
���ǽ���������⣬ajax�����ǲ��ܿ���ģ�����js��ǩ�������ǿ��Կ����<\\scrpit>      
�ǲ���ͬһ������Ҫ���������������˿ڣ�Э��  
```
var app = express();
app.use(function(req, res){
  res.jsonp({ count: 1 });
});
request(app)
.get('/?callback=something')
.expect('Content-Type', 'text/javascript; charset=utf-8')
.expect(200, /something\(\{"count":1\}\);/, done);

```
ͨ��url�е�callback�Ĳ���ȥ����������Դ

##### res.sendStatus
���÷���״̬�벢 ��״̬����Ϊ�ı�����
```
res.sendStatus = function sendStatus(statusCode) {
  var body = statusCodes[statusCode] || String(statusCode);

  this.statusCode = statusCode;
  this.type('txt');

  return this.send(body);
};
```
##### res.sendFile/sendfile
ע�⿴һ�£�����res.sendFile֮�⣬����һ��res.sendfile�ĺ������壬�ɼ��Ѿ�������ʹ��res.sendfile
```js
res.sendFile = function sendFile(path, options, callback) {
  ...
  // create file stream
  var pathname = encodeURI(path);
  var file = send(req, pathname, opts);

  // transfer
  sendfile(res, file, opts, function (err) {
    if (done) return done(err);
    if (err && err.code === 'EISDIR') return next();

    // next() all but write errors
    if (err && err.code !== 'ECONNABORTED' && err.syscall !== 'write') {
      next(err);
    }
  });
};


res.sendfile = function (path, options, callback) {
    ...
}
res.sendfile = deprecate.function(res.sendfile,
  'res.sendfile: Use res.sendFile instead');
```
�����res.sendFile�����е�����һ��[**senfile����**](https://github.com/expressjs/express/blob/4.14.1/lib/response.js#L964)������ʵ������  
```js
function sendfile(res, file, options, callback) {
  var done = false;
  var streaming;
  ...
  file.on('directory', ondirectory);
  file.on('end', onend);
  file.on('error', onerror);
  file.on('file', onfile);
  file.on('stream', onstream);
  onFinished(res, onfinish);
  ...
  // pipe
  file.pipe(res);
}
```
������������д������¼�����������¼��������ź��������ﴥ�����أ�  
���£���Դ�룬���������Ŀ�[./node_modules/send/index.js](https://github.com/pillarjs/send/blob/0.14.2/index.js#L664)��  
�����Ŀ�ͷfs.stat����ȥ��ȡ����path����ļ�������ܶ����ļ�����ô[Line676](https://github.com/pillarjs/send/blob/0.14.2/index.js#L676) �ͷ���"file"���źţ����ŵ���self.send  
�����ȡ�����ļ���Ҳ�ж����˵�next��������next��[Line687](https://github.com/pillarjs/send/blob/0.14.2/index.js#L687) �᳢�Զ��ļ�����ƴ�Ӹ��ֲ�ͬ�ĺ�׺����ȥ���Զ�ȡ  
�������и��߼�[Line 692](https://github.com/pillarjs/send/blob/0.14.2/index.js#L692)��������������ҵ���Ӧname.extension��Ŀ¼�����ļ�name.extension����ֱ�������ģ�
![sendFile.png](http://imgqi.hacking.pub/express.js%20%E6%BA%90%E7%A0%81%E4%B8%89%E6%8E%A2%20%E2%80%94%E2%80%94%20response%E7%AF%87/sendFile.png)  

Ȼ�����Line 677��Line 694���Ե�����[**self.send**](https://github.com/pillarjs/send/blob/0.14.2/index.js#L561)����������ʵ��  
```
SendStream.prototype.send = function send (path, stat) {
  ...
  // ����л���Ļ����Ͳ���Ҫ���������·���
  if (this.isConditionalGET() && this.isCachable() && this.isFresh()) {
    this.notModified()
    return
  }
  ...
  this.stream(path, opts)
}
```  
���������ô����꣬����opts�������棬����start��end�����ԣ�Ȼ��ͨ��this.stream(path, opts)�����ã�������[**this.stream��ʵ��**](https://github.com/pillarjs/send/blob/0.14.2/index.js#L737) 
```
SendStream.prototype.stream = function stream (path, options) {
  // TODO: this is all lame, refactor meeee
  var finished = false
  var self = this
  var res = this.res

  // ��������������
  var stream = fs.createReadStream(path, options)
  // ����'stream'���źţ�������response.js����file.on('stream', onstream);
  this.emit('stream', stream)
  //���ùܵ�����������res
  stream.pipe(res)

  // response finished, done with the fd
  onFinished(res, function onfinished () {
    finished = true
    destroy(stream)
  })

  // error handling code-smell
  stream.on('error', function onerror (err) {
    // request already finished
    if (finished) return

    // clean up stream
    finished = true
    destroy(stream)

    // error
    self.onStatError(err)
  })

  // ��������ᣬ����'end'
  stream.on('end', function onend () {
    self.emit('end')
  })
}
```
�ɼ���������ĵ��¼������źŶ����⣬error��end��stream��  
stream.pipe(res)ͨ���ܵ�����������res 

##### res.download 
�����������ĵĵط�Ҳ���ǵ��������res.sendFile

##### res.contentType/res.type
���÷���ͷ��content_type�ֶ�

##### res.format 
���ݿͻ��˿ɽ��ܵ�mime-type��ʽ��ִ�в�ͬ�ĺ���  
����obj����ʽ��������  
```
{
  text: function(){
    body = statusCodes[status] + '. Redirecting to ' + address;
  },

  html: function(){
    var u = escapeHtml(address);
    body = '<p>' + statusCodes[status] + '. Redirecting to <a href="' + u + '">' + u + '</a></p>';
  },

  default: function(){
    body = '';
  }
}
```
**1**.���ͻ��˽���text��ʽ�ı��ģ���ô�͵���text���Եĺ���������body  
**2**.����ͻ��˲������������ⱨ�ĸ�ʽ����ô����ִ��Obj�����default���Ժ���  
**3**.����������������û�д�������ô����״̬��406

##### res.attachment
```
res.attachment = function attachment(filename) {
  if (filename) {
    this.type(extname(filename));
  }

  this.set('Content-Disposition', contentDisposition(filename));

  return this;
};
```  
�����Content-Disposition header�ֶ��й�  
attachment ����������������ؿ�  
inline �����������Ƕ��ʾ  
�����httpͷ��ʽ����  
```
Content-disposition: inline; filename=filename.txt
Content-disposition: attachment; filename=filename.txt
```

##### res.append
��������Ƚϼ򵥣����Ǹ�����http�����ͷ�ֶ�  
�����ǰ���ù���field����ô�����value�͸���ǰvalueƴ�ӳ�������Ϊ�µ�value����������    
```
res.append('Link', ['<http://localhost/>', '<http://localhost:3000/>']);
res.append('Set-Cookie', 'foo=bar; Path=/; HttpOnly');
res.append('Warning', '199 Miscellaneous warning');
```
���������ǵ���res.set����

##### res.set/header/get/clearCookie/location/redirect
set/header(field,value) ����httpĳ�������ֶ�  
get(field) ��ȡhttpĳ��������ֶ�  
clearCookie(name) ɾ��cookie�е�ĳ���ֶΣ�������cookie���ֶΣ�����Httpͷ�ֶΣ�
location ����httpͷ��location�ֶ�
redirect 302�ض����µ�url

##### res.cookie(name, value, options)
�������ԭ�е�Set-Cookie�����¿ͻ��˵�cookie��������ӣ�ע�ⲻ�Ǹ��ǣ��µ�field-value  
����name��value�ֱ����field-value   
options�Ǿ����������ԵĶ��󣨲�ǿ��һ��Ҫ���������������Ը�ʽ����  
```
`maxage`   ����������ʱ�䣬Ҳ����ʹ��expires  
`signed`   true or false���������cookie�Ƿ���signed��cookie
`path`     Ĭ��·����"/"

����ʹ�÷�ʽ����  
httponly�Ǹ�����������cookie��������js����ű���ȡ���ģ�����Ч����xss����
res.cookie('rememberme', '1', { expires: new Date(Date.now() + 900000), httpOnly: true });
res.cookie('rememberme', '1', { maxAge: 900000, httpOnly: true })
```
�������������Ի����signed cookie
```
res.cookie = function (name, value, options) {
  ...
  var secret = this.req.secret;
  var signed = opts.signed;
  ...
  if (signed) {
    val = 's:' + sign(val, secret);
  }
  ...
  this.append('Set-Cookie', cookie.serialize(name, String(val), opts));
  ...
};
```
�ɼ�������signed cookie��value����s:��ͷ�ģ�����������ʲô��  
�����������������[����](https://github.com/expressjs/express/blob/4.14.1/test/req.signedCookies.js)  
```js
var cookieParser = require('cookie-parser')
...
app.use(cookieParser('secret'));

app.use(function(req, res){
  if ('/set' == req.path) {
    //��һ�����󷵻ص�signed cookie��Ϣ
    res.cookie('obj', { foo: 'bar' }, { signed: true });
    res.end();
  } else {
    res.send(req.signedCookies); //�ڶ��η��ش�req cookie�н�������������
  }
});

request(app)
.get('/set') //��һ������
.end(function(err, res){
  if (err) return done(err);
  var cookie = res.header['set-cookie'];

  request(app)
  .get('/')
  .set('Cookie', cookie)
  .end(function(err, res){
    if (err) return done(err); 
    res.body.should.eql({ obj: { foo: 'bar' } }); //�ڶ������󷵻ص�����
    done();
  });
});
```
���������ǿ����˽⵽�����cookeParse�⣬���ǿ��԰����ݼ��ܷ���signed cookie���棬���������Կ���ɿ����߷Ž�ȥ��  
���ͻ���Я��signed cookie�ٴ������ʱ��cookeparse�м������signed cookie�������߿���ͨ������req.signedCookies��֮ǰ�����ó�

##### res.vary
�Է���http�����е��ֶ�vary����  
vary����ֶ���Ҫ�Ǹ��߻�������������ͬһ�� URL ���Ų�ͬhttp������Դʱ����Ҫ�����ɸѡ���ʵİ汾  
�����ں���res.format������this.vary("Accept")  
������Ϊ����format����Ҫ�Բ�ͬ��ʽ��Accpet���ز�ͬ��Content-Type��ʽ  
```
Accept	��֪���������ͺ���ý������	Content-Type
Accept-Language	��֪���������ͺ�������	Content-Language
Accept-Charset	��֪���������ͺ����ַ���	Content-Type
Accept-Encoding	��֪���������ú���ѹ����ʽ	Content-Encoding
```
��Ȼ������ǵ���Դ���ڲ�ͬ������������ǲ�һ���ģ�Ҳ�����������UA  
��ô���Ƿ���������http��Ӧ�ú����������ֶ�
```
Vary: User-Agent, Cookie
```

##### res.render
���������Ҫ������app.render���������application.js��ʱ���ٽ���

