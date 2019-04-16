---
layout: simple-article
title: Koa源码解读
date: 2018-04-02 21:15:37
categories:
    - 服务器端
    - Node.js
tags:
    - Koa
---
### 前记
![onion](https://qingmooc-v1.oss-cn-qingdao.aliyuncs.com/other/onion.png)

### 1. Koa基本用法

首先咱们先看下Koa框架的基本用法

```
const Koa = require('koa');

const app = new Koa();
app.use(async (ctx, next) => {
  const oldTime = +new Date();
  await next();
  const newTime = +new Date();
  const interval =  newTime - oldTime;
  console.log(interval);
});
app.use(async (ctx, next) => {
  ctx.body = "hello";
  await next();
});
app.listen(3000);
```

`ctx` ，顾名思义，是app运行上下文的意思，里面绑定了request和response的信息，具备可以描述一次请求的所有信息；`next`就很神奇了，**每调用一次next，就会进入下一个app.use的函数，直到调用到执行完最后一个函数，又会返回之前上一层调用next函数的地方，继续执行上一层函数接下来的部分，这样以此类推，直到运行完所有的app.use中的函数**。如果不太明白，可以看下洋葱图，洋葱图的每一层就是一个app.use中的函数，这个函数我们给它个名称，叫做 **`middleware(中间件)`**。

至于怎样达到上面的效果，让我们从Koa源码来看看。

### 2. Koa源码 

#### 1. 首先看`package.json`

我们只需要找到入口文件就好了。
```
...
"main": "lib/application.js",
...
```
入口文件确定了 `lib/application.js`

#### 2. 接着翻到`lib/application.js`

```
...
module.exports = class Application extends Emitter {

  constructor() {
    super();

    this.proxy = false;
    this.middleware = [];
    this.subdomainOffset = 2;
    this.env = process.env.NODE_ENV || 'development';
    this.context = Object.create(context);
    this.request = Object.create(request);
    this.response = Object.create(response);
    if (util.inspect.custom) {
      this[util.inspect.custom] = this.inspect;
    }
  }
...
}
```
在我们执行 `const app = new Koa();`时，调用的就是以上的constructor()。

可见，Application继承自Emitter，所以它也有 Emitter 的方法，比如 emit, once 等等

==构造函数中构建了`context request response`三个对象，还有一个middleware数组==，这四个变量非常重要，可以说整个Koa框架就是围绕着这四个变量。

#### 3. `lib/application.js`中可以看到`const context = require('./context');`，所以我们翻到`lib/context.js`看看
```
const delegate = require('delegates');
...
const proto = module.exports = {

  ...
  get cookies() {
    if (!this[COOKIES]) {
      this[COOKIES] = new Cookies(this.req, this.res, {
        keys: this.app.keys,
        secure: this.request.secure
      });
    }
    return this[COOKIES];
  },

  set cookies(_cookies) {
    this[COOKIES] = _cookies;
  }
};
...
// 将response方法委托到 context
delegate(proto, 'response')
  .method('attachment')
  .method('redirect')
  .method('remove')
  .method('vary')
  .method('set')
  .method('append')
  .method('flushHeaders')
  .access('status')
  .access('message')
  .access('body')
  .access('length')
  .access('type')
  .access('lastModified')
  .access('etag')
  .getter('headerSent')
  .getter('writable');

// 将request方法委托到 context
delegate(proto, 'request')
  .method('acceptsLanguages')
  .method('acceptsEncodings')
  .method('acceptsCharsets')
  .method('accepts')
  .method('get')
  .method('is')
  .access('querystring')
  .access('idempotent')
  .access('socket')
  .access('search')
  .access('method')
  .access('query')
  .access('path')
  .access('url')
  .access('accept')
  .getter('origin')
  .getter('href')
  .getter('subdomains')
  .getter('protocol')
  .getter('host')
  .getter('hostname')
  .getter('URL')
  .getter('header')
  .getter('headers')
  .getter('secure')
  .getter('stale')
  .getter('fresh')
  .getter('ips')
  .getter('ip');
```
> 对cookies的操作使用了 getter setter的写法 https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Functions/get

这里，他用了一个叫做`delegates`的库，将proto和request还有response做了一个委托的操作。具体我们来看看`delegate`的代码

#### 4. `delegate`库
```
function Delegator(proto, target) {
  if (!(this instanceof Delegator)) return new Delegator(proto, target);
  this.proto = proto;
  this.target = target;
  this.methods = [];
  this.getters = [];
  this.setters = [];
  this.fluents = [];
}
```
`delegate(proto, 'request')`调用的就是上面这个函数，这里 target 就是 'request'。
```
Delegator.prototype.method = function(name){
  var proto = this.proto;
  var target = this.target;
  this.methods.push(name);

  proto[name] = function(){
    return this[target][name].apply(this[target], arguments);
  };

  return this;
};
```

`.method('acceptsLanguages')`调用的就是上面的函数，这里 name 就是 'acceptsLanguages'

这个函数主要做的事就是当调用 proto 对象的名为 name 的方法时，将去 target 中寻找这个方法，然后调用。

> JS apply 函数 https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Function/apply

```
Delegator.prototype.getter = function(name){
  var proto = this.proto;
  var target = this.target;
  this.getters.push(name);

  proto.__defineGetter__(name, function(){
    return this[target][name];
  });

  return this;
};

Delegator.prototype.setter = function(name){
  var proto = this.proto;
  var target = this.target;
  this.setters.push(name);

  proto.__defineSetter__(name, function(val){
    return this[target][name] = val;
  });

  return this;
};

Delegator.prototype.access = function(name){
  return this.getter(name).setter(name);
};
```
`.getter('origin')` 和 `.access('querystring')`调用的是上面的函数，与`.method`类似，也是调用proto对象的 getter 或者 accessor 时，在 target 中寻找对应的 getter 或者 accessor 
> JS __defineGetter__方法 https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object/__defineGetter__

至此，**就明白`lib/context.js`中那两大串函数是干嘛的了，就是将 request 和 response 的方法或者getter/setter委托到 context 中，起到的作用就是可以用多个方法访问一个属性或者方法**，比如 `ctx.acceptsEncodings()` 和 `ctx.request.acceptsEncodings()`，的效果是完全一模一样的。

#### 5. 再看`lib/request.js` `lib/response.js`

```
// lib/request.js
module.exports = {

  get header() {
    return this.req.headers;
  },

  set header(val) {
    this.req.headers = val;
  },
  ...
}

// lib/response.js
module.exports = {

  get socket() {
    return this.res.socket;
  },
  ...
}
```
这两个文件还是比较长的，但是里面全是对 request 和 response 的 getter, setter 还有 method 的定义，还是很容易懂。

值得注意的是，从上面代码可以看到，属性都是大部分都是从`this.req`或者`this.res`中取来的，而在这两个文件中并不能找到`this.res`或者`this.req`的声明，说明`res`和`req`都是之后才绑定到 request 或者 response上的。

#### 6. 知道了context request response 之后，再看 application的use, listen方法
```
  ...
  use(fn) {
    if (typeof fn !== 'function') throw new TypeError('middleware must be a function!');
    if (isGeneratorFunction(fn)) {
      deprecate('Support for generators will be removed in v3. ' +
                'See the documentation for examples of how to convert old middleware ' +
                'https://github.com/koajs/koa/blob/master/docs/migration.md');
      fn = convert(fn);
    }
    debug('use %s', fn._name || fn.name || '-');
    this.middleware.push(fn);
    return this;
  }
  
  listen(...args) {
    debug('listen');
    const server = http.createServer(this.callback());
    return server.listen(...args);
  }
  ...
```
可见，`use`就是将 fn 加入到 middleware 数组中，并且确认 fn 一定为函数。这里还判断了下 fn 是否为 generator，如果是的话，将 fn 利用 koa-convert 来转换下，这个涉及到异步编程的问题，这里先不详细讲，以后的博文会提到。

`listen中`将`this.callback()`作为参数传入，根据下面node源码片段(来自`lib/_http_server.js`)可以知道，当Server收到一个请求，就会触发 'request' 事件，也就会调用 requestListener ，这里就是 `this.callback()`

```
function Server(requestListener) {
  if (!(this instanceof Server)) return new Server(requestListener);
  net.Server.call(this, { allowHalfOpen: true });

  if (requestListener) {
    this.addListener('request', requestListener);
  }
  ...
}
```

#### 7. 再看application中的callback()
```
  ...
  callback() {
    const fn = compose(this.middleware);

    if (!this.listenerCount('error')) this.on('error', this.onerror);
    
    // req res是在触发'request'事件后，传递的参数
    const handleRequest = (req, res) => {
      const ctx = this.createContext(req, res);
      return this.handleRequest(ctx, fn);
    };

    return handleRequest;
  }
  ...
```

1. 首先执行的是`this.createContext(req, res)`，那就先看`createContext函数`，==注意这里的req，res参数里面已经有具体的请求参数了==

```
  ...
  createContext(req, res) {
    const context = Object.create(this.context);
    const request = context.request = Object.create(this.request);
    const response = context.response = Object.create(this.response);
    context.app = request.app = response.app = this;
    context.req = request.req = response.req = req;
    context.res = request.res = response.res = res;
    request.ctx = response.ctx = context;
    request.response = response;
    response.request = request;
    context.originalUrl = request.originalUrl = req.url;
    context.state = {};
    return context;
  }
  ...
```
到这里就能看出来了，就是在这里将`req res app`绑在 request response 和 context上，request和response 还有context 也互相绑定。


2. 绑定后，调用了`this.handleRequest(ctx, fn);`，fn是由`const fn = compose(this.middleware);`生成的，所以这里看下`koa-compose`干了什么
```
function compose (middleware) {
  // 参数检查
  if (!Array.isArray(middleware)) throw new TypeError('Middleware stack must be an array!')
  for (const fn of middleware) {
    if (typeof fn !== 'function') throw new TypeError('Middleware must be composed of functions!')
  }
  return function (context, next) {
    // last called middleware #
    let index = -1
    return dispatch(0)
    
    function dispatch (i) {
      if (i <= index) return Promise.reject(new Error('next() called multiple times'))
      index = i
      let fn = middleware[i]
      if (i === middleware.length) fn = next
      if (!fn) return Promise.resolve()
      try {
        // 返回一个Promise.resolve 
        // fn 指的是middleware中一个一个的函数
        // fn 的第一个参数是context
        // fn的第二个参数是middleware数组中它之后的函数
        // 这下知道为啥 koa的中间件，参数都是 (ctx, next) 了吧
        return Promise.resolve(fn(context, dispatch.bind(null, i + 1)));
      } catch (err) {
        return Promise.reject(err)
      }
    }
  }
}
```
关键就是`return Promise.resolve(fn(context, dispatch.bind(null, i + 1)));`，这是一个递归。

middleware数组中是app.use(fn)中的fn组成的数组。dispatch(0) 会取出middleware数组的第一个fn，然后将 context 作为第一个参数，将 `dispatch.bind(null, 1)`作为第二个参数传入 fn。`dispatch.bind(null, 1)`会取出middleware数组中的第二个fn，然后将`dispatch.bind(null, 2)`做为第二个参数，传给fn，以此类推。直到将 middleware 数组的函数都取一遍，也就是说 `i === middleware.length`，这时会返回一个 `Promise.resolve()`

==fn 的第一个参数是context，fn的第二个参数是middleware数组中它之后的函数
，这下知道为啥 koa的中间件，参数都是 (ctx, next) 了吧==。

在fn中，如果有`await next()`，这样的语句，那么就会直接调用下一层函数，下一层函数再执行`await next()`,就会继续下一层，直到最后一层，最后的是fn是`Promise.resolve()`，它没有返回值，然后返回倒数第二层，继续执行没执行完的部分，以次类推

返回的像是一个函数洋葱，第一层是middleware中第一个函数，第一层调用next进入第二层函数，第二层调用next进入第三个...以此类推

**==有以下几点需要注意==**：

* `return function (context, next) {...}` 中的next参数根本没有用，因为在之后的调用中可以看到，这个参数根本没传，所以是undifined，但是还好有
```
if (i === middleware.length) fn = next; // fn = undifined
if (!fn) return Promise.resolve() 
```
这样就会在函数洋葱的最后补上一个Promise.resolve();

* 一个fn内只能有一个`await next()`，也就是说只能调用下一个fn一次，如果调用两次，会导致报出`Error: next() called multiple times`错误

> 在同一个中间件中多次调用 next() （执行next就是执行dispatch）的时候，index 会大于 i，从代码上看，每个中间件都有一个自己的作用域，也就是说同一个中间件，i 是不变的，在 i 不变的情况下，多次调用 next的情况下，第一次调用，index 小于 i，第二次调用，index就等于 i了，然后就会报错了

* 之所以返回Primose.resolve()而不是Promise，是因为中间件函数不止支持 async 函数，还支持普通函数。

#### 8. 再看 `this.handleRequest(ctx, fn)`, fn也就是上面`compose`的函数了
```
  handleRequest(ctx, fnMiddleware) {
    const res = ctx.res;
    res.statusCode = 404;
    const onerror = err => ctx.onerror(err);
    const handleResponse = () => respond(ctx);
    onFinished(res, onerror);
    return fnMiddleware(ctx).then(handleResponse).catch(onerror);
  }
```

`fnMiddleware(ctx).then(handleResponse).catch(onerror)`这句话是核心，fnMiddleware就是之前返回的函数的火车；先执行这一系列的函数，这里面可能有解析参数啊，处理请求啊，等等操作，然后再执行 `handleResponse`。handleResponse就是调用了respond函数

#### 9. 再看respond函数
```
function respond(ctx) {
  // allow bypassing koa
  if (false === ctx.respond) return;

  const res = ctx.res;
  if (!ctx.writable) return;

  let body = ctx.body;
  const code = ctx.status;

  // ignore body
  if (statuses.empty[code]) {
    // strip headers
    ctx.body = null;
    return res.end();
  }

  if ('HEAD' == ctx.method) {
    if (!res.headersSent && isJSON(body)) {
      ctx.length = Buffer.byteLength(JSON.stringify(body));
    }
    return res.end();
  }

  // status body
  if (null == body) {
    body = ctx.message || String(code);
    if (!res.headersSent) {
      ctx.type = 'text';
      ctx.length = Buffer.byteLength(body);
    }
    return res.end(body);
  }

  // responses
  if (Buffer.isBuffer(body)) return res.end(body);
  if ('string' == typeof body) return res.end(body);
  if (body instanceof Stream) return body.pipe(res);

  // body: json
  body = JSON.stringify(body);
  if (!res.headersSent) {
    ctx.length = Buffer.byteLength(body);
  }
  res.end(body);
}
```
==可以看出，就是根据操作改变后的ctx，来规整出一个response==

#### 10. 最后是错误处理的onerror函数
```
  onerror(err) {
    if (!(err instanceof Error)) throw new TypeError(util.format('non-error thrown: %j', err));

    if (404 == err.status || err.expose) return;
    if (this.silent) return;

    const msg = err.stack || err.toString();
    console.error();
    console.error(msg.replace(/^/gm, '  '));
    console.error();
  }
```

### 3. 总结
Koa 官网对 Koa 有如下描述：
> koa 是由 Express 原班人马打造的，致力于成为一个更小、更富有表现力、更健壮的 Web 框架。 使用 koa 编写 web 应用，通过组合不同的 generator，可以免除重复繁琐的回调函数嵌套， 并极大地提升错误处理的效率。koa 不在内核方法中绑定任何中间件， 它仅仅提供了一个轻量优雅的函数库，使得编写 Web 应用变得得心应手。

可以说是非常贴切了，Koa的实现非常简洁，源码加起来不过两千行，但是借助中间件模型保持了极大的可拓展性，只要用各种不同的 middleware 一层一层的搭建就能实现多种多样的功能，感叹一句，TJ大神实在是强。