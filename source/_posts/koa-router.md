---
title: koa-router包浅析
date: 2018-03-25 22:17:37
categories:
    - 服务器端
    - Node.js
    - Koa
tags:
    - Koa
---
启一个web服务器，路由是一个很重要的地方，根据客户端访问的url与方法，调用不同的接口，Koa 中使用的是 `koa-router`包，这里就看一下它的原理。
<!-- more -->
### 一. 基本用法
```
const Koa = require('koa');
const Router = require('koa-router');

const router = new Router();
router.get('/api/test', async (ctx, next) => {
  ctx.body = 'test';
  await next();
});

app.use(router.routes());

app.listen(3000);
```

### 二. 源码
`main` 为 `lib/router.js`，咱们就从这个开始看
#### 1. Router类
```
...
module.exports = Router;

function Router(opts) {
  if (!(this instanceof Router)) {
    return new Router(opts);
  }

  this.opts = opts || {};
  this.methods = this.opts.methods || [
    'HEAD',
    'OPTIONS',
    'GET',
    'PUT',
    'PATCH',
    'POST',
    'DELETE'
  ];

  this.params = {};
  this.stack = [];
};


```
当调用`const router = new Router()`时，就会调用上面的Router构造函数。

这里可以传入一个opts对象作为参数，具体有以下属性：
* methods: 允许的HTTP方法，默认是`    'HEAD',
    'OPTIONS',
    'GET',
    'PUT',
    'PATCH',
    'POST',
    'DELETE'`数组
* prefix: 路由前缀

#### 2. 绑定方法
```
var methods = require('methods');
...

methods.forEach(function (method) {
  Router.prototype[method] = function (name, path, middleware) {
    var middleware;

    if (typeof path === 'string' || path instanceof RegExp) {
      middleware = Array.prototype.slice.call(arguments, 2);
    } else {
      middleware = Array.prototype.slice.call(arguments, 1);
      path = name;
      name = null;
    }

    this.register(path, [method], middleware, {
      name: name
    });

    return this;
  };
});
```

之后，`methods`包会返回HTTP方法，`acl
bind
checkout
connect
copy
delete
get
head
link
lock
m-search
merge
mkactivity
mkcalendar
mkcol
move
notify
options
patch
post
propfind
proppatch
purge
put
rebind
report
search
source
subscribe
trace
unbind
unlink
unlock
unsubscribe`真的是非常多，大部分可能都不会用到吧。。。

然后为每个HTTP方法都声明一个Router的同名的原型方法，这样每次调用 `router.get`等方法时，实际上就调用的是这个函数。

再看`Router.prototype[method]`的函数体。可以看到，**它接收三个参数，第一个参数`name`，其实是可选的，之后的`path`是必须的，最后的 `middleware` 是不限数量的，是函数组成的数组**。

取完参数后，调用了`this.register`方法

```
var Layer = require('./layer');
...
Router.prototype.register = function (path, methods, middleware, opts) {
  opts = opts || {};

  var stack = this.stack;

  // create route
  var route = new Layer(path, methods, middleware, {
    end: opts.end === false ? opts.end : true,
    name: opts.name,
    sensitive: opts.sensitive || this.opts.sensitive || false,
    strict: opts.strict || this.opts.strict || false,
    prefix: opts.prefix || this.opts.prefix || "",
  });
  ...
  return route;
};
```
这里创建了一个 Layer 对象，我们看看这是啥

#### 3. `Layer`类
```
var pathToRegExp = require('path-to-regexp');

module.exports = Layer;

function Layer(path, methods, middleware, opts) {
  this.opts = opts || {};
  this.name = this.opts.name || null;
  this.methods = [];
  this.paramNames = [];
  this.stack = Array.isArray(middleware) ? middleware : [middleware];

  methods.forEach(function(method) {
    var l = this.methods.push(method.toUpperCase());
    if (this.methods[l-1] === 'GET') {
      this.methods.unshift('HEAD');
    }
  }, this);

  // ensure middleware is a function
  this.stack.forEach(function(fn) {
    var type = (typeof fn);
    if (type !== 'function') {
      throw new Error(
        methods.toString() + " `" + (this.opts.name || path) +"`: `middleware` "
        + "must be a function, not `" + type + "`"
      );
    }
  }, this);

  this.path = path;
  this.regexp = pathToRegExp(path, this.paramNames, this.opts);

  debug('defined route %s %s', this.methods, this.opts.prefix + this.path);
};

```

* `methods`中是`methods`中的方法都转成大写形式，特别的，如果有GET方法，就把HEAD方法也加入(因为HEAD方法基本上就是不返回报体的GET的方法)。
* `stack` 中是 middleware 函数组成的数组，这里做了下参数检查，防止执行出错。

最后是`this.regexp` ，这个调用了 [`path-to-regexp`包](https://www.npmjs.com/package/path-to-regexp)，这里就不详细讲了，只需要知道这里返回的是一个Regexp实例，可以根据这个正则对象来匹配路径，`this.paramNames`是一个数组，是根据路由中的站位符分析出的这个路由拥有的 param 参数。

#### 4. `Router.prototype.register`方法

我们接着看 `this.register`方法的剩余部分

```
Router.prototype.register = function (path, methods, middleware, opts) {
  opts = opts || {};

  var stack = this.stack;

  // create route
  var route = new Layer(path, methods, middleware, {
    end: opts.end === false ? opts.end : true,
    name: opts.name,
    sensitive: opts.sensitive || this.opts.sensitive || false,
    strict: opts.strict || this.opts.strict || false,
    prefix: opts.prefix || this.opts.prefix || "",
  });

  if (this.opts.prefix) {
    route.setPrefix(this.opts.prefix);
  }
  
  // add parameter middleware
  Object.keys(this.params).forEach(function (param) {
    route.param(param, this.params[param]);
  }, this);

  // register route with router
  if (methods.length || !stack.length) {
    // if we don't have parameters, put before any with same route
    // nesting level but with parameters
    var added = false;

    if (!route.paramNames.length) {
      var routeNestingLevel = route.path.toString().split('/').length;

      added = stack.some(function (m, i) {
        var mNestingLevel = m.path.toString().split('/').length;
        var isParamRoute = !!m.paramNames.length;
        if (routeNestingLevel === mNestingLevel && isParamRoute) {
          return stack.splice(i, 0, route);
        }
      });
    }

    if (!added) stack.push(route);
  } else {
    stack.some(function (m, i) {
      if (!m.methods.length && i === stack.length - 1) {
        return stack.push(route);
      } else if (m.methods.length) {
        if (stack[i - 1]) {
          return stack.splice(i, 0, route);
        } else {
          return stack.unshift(route);
        }
      }
    });
  }

  return route;
};
```
这里就是将每个路由(Layer对象)都push进this.stack数组。这里还进行了嵌套路由的处理。
比如执行完
```
router.get('/api/test/:id', async (ctx, next) => {
  const { id } = ctx.request.query;
  ctx.body = id;
  await next();
});

router.post('/api/test', async (ctx, next) => {
  ctx.body = 'ok';
  await next();
});
```
`this.stack`中就是
```
[{
  opts: {
    end: true,
    name: null,
    sensitive: false,
    strict: false,
    prefix: ''
  },
  name: null,
  methods: ['HEAD', 'GET'],
  paramNames: [[Object]],
  stack: [[AsyncFunction]],
  path: '/api/test/:id',
  regexp: { /^\/ api\/test\/((?:[^\/]+?))(?:\/(?=$))?$/i keys: [Array] } 
},
{
  opts: {
    end: true,
    name: null,
    sensitive: false,
    strict: false,
    prefix: ''
  },
  name: null,
  methods: ['POST'],
  // 上述定义的 post路由没有 query 参数
  paramNames: [],
  // middleware 数组
  stack: [[AsyncFunction]],
  path: '/api/test',
  regexp: { /^\/api\/test(?:\/(?=$))?$/i keys: [] }
}];
```
可见，通过stack中的每个 Layer 对象，就可以匹配一个路由，然后执行stack中的中间件函数。

#### 5. `Router.prototype.routes`方法

```
Router.prototype.routes = Router.prototype.middleware = function () {
  var router = this;

  var dispatch = function dispatch(ctx, next) {
    debug('%s %s', ctx.method, ctx.path);

    var path = router.opts.routerPath || ctx.routerPath || ctx.path;
    var matched = router.match(path, ctx.method);
    var layerChain, layer, i;

    if (ctx.matched) {
      ctx.matched.push.apply(ctx.matched, matched.path);
    } else {
      ctx.matched = matched.path;
    }

    if (!matched.route) return next();

    layerChain = matched.pathAndMethod.reduce(function(memo, layer) {
      memo.push(function(ctx, next) {
        ctx.captures = layer.captures(path, ctx.captures);
        ctx.params = layer.params(path, ctx.captures, ctx.params);
        return next();
      });
      return memo.concat(layer.stack);
    }, []);

    return compose(layerChain)(ctx, next);
  };

  dispatch.router = this;

  return dispatch;
};

Router.prototype.match = function (path, method) {
  var layers = this.stack;
  var layer;
  var matched = {
    path: [],
    pathAndMethod: [],
    route: false
  };

  for (var len = layers.length, i = 0; i < len; i++) {
    layer = layers[i];

    debug('test %s %s', layer.path, layer.regexp);

    if (layer.match(path)) {
      matched.path.push(layer);

      if (layer.methods.length === 0 || ~layer.methods.indexOf(method)) {
        matched.pathAndMethod.push(layer);
        if (layer.methods.length) matched.route = true;
      }
    }
  }

  return matched;
};

Layer.prototype.match = function (path) {
  return this.regexp.test(path);
};
```
由于是`app.use(router.routes())`的用法，所以这个函数返回的参数形式，必然符合中间件函数的要求。

这个函数就是将`ctx.path`路径用`stack`中每个`Layer`中的正则表达式去匹配路径，并且将解析出的 url params 赋给 `ctx.params`，这样之后执行的中间件函数中就可以通过`ctx.params`参数来访问url 参数了。最后用 koa-compose将中间件数组连起来调用。

**路由就是根据url然后决定调用哪些后台方法**。**koa-router将每个路由都抽象成一个`Layer`，`Layer`中拥有可以匹配路径的正则和匹配成功后将要运行的middlleware数组，把所有`Layer`都放在内部的`stack`数组中，当来了新请求，就用`Layer`中的正则去匹配，匹配成功了，就执行`Layer`中的middleware数组。这就完成了路由**。


#### 6. 其他方法 

1. `router.use`方法

用法:
```
// 针对所有中间件，增加某个中间件
router
  .use(session())
  .use(authorize());
 
// 针对对某个路由，增加某个中间件
router.use('/users', userAuth());
 
// 针对某一系列路由，增加某个中间件
router.use(['/users', '/admin'], userAuth());
 
app.use(router.routes());
```

以上就是`use`方法的用处。不得不说，我其实是很少用到这个方法的，因为第一个用处，完全可以在调用`app.use(router.routes())`之前调用`app.use(someMiddleware())`来完成。

第二个用法，一般来说，可以直接将那个中间件的逻辑写进`router.get('/users', userAuth(), otherMiddleware())`，这样就可以了，除非是你想针对这个路由的所有方法(get post put delete...)，都执行这个中间件，那的确这样写方便些。我建议**可以运行一些通用的路由中间件**，比如保存路由日志`router.use(saveLog())`

第三个用法跟第二个方法差不多。

剩下的方法我几乎不用了，这里就不提了


### 三. 总结

[koa-router](https://github.com/alexmingoia/koa-router#readme) 是一个蛮基础的包，基本上用到了koa框架就会用到这个包。

回顾整个这个包整个路由的过程，有两个包是最重要的`path-to-regexp`和`koa-compose`，第一个包可以返回一个可以匹配特定路由的正则对象，第二个包可以将中间件函数连起来(详见[[Node.js] 十步完成Koa2框架源码阅读](https://github.com/LouisWT/Blog/issues/1))。有兴趣的可以仔细研究下。