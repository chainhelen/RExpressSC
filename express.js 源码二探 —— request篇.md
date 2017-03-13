#### ����ƪ��̽
��ƪд�ĺ��ң�ֻ�Ǹ�����ʱ����д�ġ���Ҫ��ͼ��app.use��app.[method]֮���������֣�������express���м����·�ɵ�ʵ���߼���  

#### ��ƪ
��ƪ���п�����Ȼ������ƪ�ķ�ɢ��˼ά�����Ը�λ���߿���ҪС���Ķ��������²۶�����

#### ����
��������ƪһ�£�Դ��汾��[express.js 4.14.1](https://github.com/expressjs/express/tree/4.14.1)

----  
## requestԴ��  
��[request.js](https://github.com/expressjs/express/blob/4.14.1/lib/request.js)���������¿�����

##### IncomingMessage
```js
var req = exports = module.exports = {
  __proto__: http.IncomingMessage.prototype
};
```
[������������](https://github.com/expressjs/express/blob/4.14.1/lib/request.js#L31)   
����Ҫ**����request������󣬸�http.IncomingMessage��ʲô��ϵ�������ǹ���request��\_\_proto\_\_ԭ������**  
����֪��nodejs�ṩ��ԭ��http����ͨ������ô�õ�
```
var http = require('http');
http.createServer( (function(req, res) { 
    res.end("hello world");
}) ).listen("3000");
```  
��ô���ǿ���nodejs��Դ���������http.js���ҵ�[����Ĵ���](https://github.com/nodejs/node/blob/v6.9.4/lib/http.js#L8)  
```
const server = require('_http_server');
...
const Server = exports.Server = server.Server;

exports.createServer = function(requestListener) {
  return new Server(requestListener);
};
```
������Կ����ǻص������Ǵ�\_http\_server.js��������ģ���ô������[\_http\_server.js](https://github.com/nodejs/node/blob/v6.9.4/lib/_http_server.js#L229)�Ĵ��룬����
```
function Server(requestListener) {
  ...
  if (requestListener) {
    this.addListener('request', requestListener);
  }
  ...
}
```
���Կ����ǹ۲���ģʽ����ô����\_http\_server.js�������ؼ��� "emit('request'"��Ѹ���ҵ�����Ĵ�������
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
���Կ������req�Ǵ����ﴫ����ȥ�ģ����濴��[380��](https://github.com/nodejs/node/blob/v6.9.4/lib/_http_server.js#L463)
```
function onParserExecuteCommon(ret, d) {
    ...
    var req = parser.incoming;
    ...
}
```
��Ȼ���кöദ���Կ���var req = parser.incoming;��ôȥ����incoming��ô���ģ�ȥ��\_http\_commong.js [60��](https://github.com/nodejs/node/blob/v6.9.4/lib/_http_common.js#L60)
```
parser.incoming = new IncomingMessage(parser.socket);
```
��ô����֪������ʵrequest���ǹ�����IncomingMessage new ������һ������  
����Ҳ�ͽ���������express��reqΪʲôҪ��ԭ�����ϼ���**IncomingMessage**  


##### req.get && req.header
��һ���֣����Ƿ���httpͷ���ֶΣ��Ƚ�����˼�ľ���referer  
�Ǹ��淶http���ߵ����ƴ����referer�����Ե�������referrer��referer������ͬһ����˼    
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
����Ǹ�accepts�⣬��Ҫ������֤http����Դ����  
���ƥ�䷵��true���෴�򷵻�undefined����ʱ��responseӦ�÷���406  
���������ļ���accepts����һһ������  

##### req.range
���ص�httpͷ�п���Я��range�ֶΣ�ָ����Ҫ������Դ����һ����
```
Range: bytes=500-999
```
�����Դ�ܵĴ�С��10000��������������http 200����ôres��ͷ����Ӧ����������  
```
Content-Range: bytes 500-999/10000
```

��req.range������������Ǹ�ʲô���أ�
```
req.range = function range(size, options) {
  var range = this.get('Range');
  if (!range) return;
  return parseRange(size, range, options);
};
```
����һ��ע�⼰����Դ��:  

* ���req��httpͷ��Range�ֶ�δ�������ú�������undefined
* ��������Դ���ܻ��ֳ�size��Χ�ڵģ��ú�������-1
* ����﷨���󣬸ú�������-2
* �����ķ��أ�������
```
Range: bytes=500-600,601-999 //httpͷRange�ֶ�

//��ô�ú�������
[
    {"start" : 500, "end" : 600},
    {"start" : 601, "end" : 999}
]
//���ҷ��ص����������һ������type = bytes, ����ֵΪbytes����Ϊ����ͷRange�������"bytes" 
```

##### req.param
����Ƚϼ򵥣� param(name, defaultValue)������req.params��req.body��req.query�ȶ�������ȡ���name������ֵ��������Ҳ������Ǿͷ���defaultValue    
req.params��body��query�������ֱ�ָ���������²���

 *  ·�ɵ�ռλ: _/user/:id_
 *  body���������: id=12, {"id":12}
 *  url����Ĳ���: ?id=12

##### req.is  
�����ж������http�������ͣ�ֱ���ú���ע�����Ӿͺ�
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
ʣ�µĺ�������get������  

* protocol  //http����https
* secure   //����true����false����ȫ����httpsЭ��
* ip     //����ip��ַ
* ips    //ip��ַ�������м�����ip
* subdomains  //���������飬tobi.ferrets.example.com���� ["ferrets", "tobi"]
* path  //url�����·��
* hostname //������
* host  //����һ��������Ҫ������
* fresh // ����Last-Modified��ETag�жϿͻ����Ƿ������»��棬û�еĻ�˳�ֻ���һ��
* stale //ͬ�ϣ������жϵ��ǿͻ��˻����Ƿ����
* xhr //�ж��Ƿ���xmlhttprequest��ajax�����͵�����

��������defineProperty���get�������������Է������Ե�ʱ��ᴥ����������Щ����д������û��set������

##### ���
��ônodejs http.createServer����Ĳ���req������ô�ͺ��������Ƕ����req��������ȥ���أ�    
���м��/lib/middleware/init.js
```
exports.init = function(app){
  return function expressInit(req, res, next){
    ... 
    //ͨ��ԭ����������req�����
    req.__proto__ = app.request;
    res.__proto__ = app.response;
    ...
  };
};
```
