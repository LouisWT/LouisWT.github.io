---
title: Passport.js 源码阅读
date: 2018-04-19 20:12:37
categories:
    - 服务器端
    - Node.js
tags:
    - Koa

---
对一个网站应用而言，鉴权是非常重要的一点，[`passport.js`](http://www.passportjs.org/docs/downloads/html/)就是为了解决这个问题诞生的。它抽象出了策略这个概念，可以根据不同的接口调用不同的鉴权策略。
<!-- more -->

passport.js支持多种鉴权方式：
* username+password形式
* [OpenID鉴权](https://www.biaodianfu.com/learn-openid.html)
* [OAuth鉴权](http://www.ruanyifeng.com/blog/2014/05/oauth_2_0.html)
* 使用基础的策略包编写自己的策略

### 一. 基本使用

```
const Koa = require('koa');
const Router = require('koa-router');
const passport = require('koa-passport');
const CustomStrategy = require('passport-custom').Strategy;

const app = new Koa();
const router = new Router();

passport.use('custom', new CustomStrategy((ctx, done) => {
  const { authorization } = ctx.headers;
  if (authorization === 'token') {
    return done(null, { user: 1 });
  } else {
    done(null, false);
  }
}));

const initialize = () => {
  return passport.initialize();
};  

app.use(initialize());

router.get('/', passport.authenticate('custom', {
  session: false
}), async (ctx, next) => {
  ctx.body = 'hello';
  await next();
});

app.use(router.routes());

app.listen(3000);
```
注意，**由于passport本身是针对express的，返回的中间件函数都是`req, res, next`形式**，不能直接用于Koa框架，所以我们用的是`koa-passport`包，但是这个包其实还是依赖于 `passport`包，只是重写了一些方法，使它可以用于Koa框架。我们还是看`passport`包的代码，只是看到`res req`时自动替换成`ctx.request ctx.response`就好了

### 二. `passport.js`源码

`main`为`./lib/index.js`

```
var Passport = require('./authenticator')
exports = module.exports = new Passport();
```
所以我们从`lib/authenticator`开始看

#### 1. `authenticator.js`代码

```
module.exports = Authenticator;

function Authenticator() {
  this._key = 'passport';
  this._strategies = {};
  this._serializers = [];
  this._deserializers = [];
  this._infoTransformers = [];
  this._framework = null;
  this._userProperty = 'user';
  
  this.init();
}

Authenticator.prototype.init = function() {
  this.framework(require('./framework/connect')());
  this.use(new SessionStrategy(this.deserializeUser.bind(this)));
  this._sm = new SessionManager({ key: this._key }, this.serializeUser.bind(this));
};
```
`init`方法调用了`framework`和`use`方法

1. 先看`framework`方法

```
Authenticator.prototype.framework = function(fw) {
  this._framework = fw;
  return this;
};

// ./framework/connect
var initialize = require('../middleware/initialize')
  , authenticate = require('../middleware/authenticate');

exports = module.exports = function() {
  
  // HTTP extensions.
  exports.__monkeypatchNode();
  
  return {
    initialize: initialize,
    authenticate: authenticate
  };
};
```
`framework`方法就是对`this._framework`的初始化方法，经过这步之后，`this._framework`就有 `initialize` 和 `authenticate` 这两个方法了，这里先不管，之后调用的时候再看。

2. 再看`this.use(new SessionStrategy(this.deserializeUser.bind(this)))`
```
var SessionStrategy = require('./strategies/session');

Authenticator.prototype.use = function(name, strategy) {
  if (!strategy) {
    strategy = name;
    name = strategy.name;
  }
  if (!name) { throw new Error('Authentication strategies must have a name'); }
  
  this._strategies[name] = strategy;
  return this;
};

// ./strategies/session
function SessionStrategy(options, deserializeUser) {
  if (typeof options == 'function') {
    deserializeUser = options;
    options = undefined;
  }
  options = options || {};
  
  Strategy.call(this);
  this.name = 'session';
  this._deserializeUser = deserializeUser;
}
```
这个应该就是初始化添加一个默认的策略，这个我感觉没啥用，因为一般都不会用它自带的session策略。

**主要是use方法，这个方法在我们添加策略时也用到了**，如调用`passport.use('custom', new CustomStrategy(...))`时，就在`this._strategy`对象添加一个键值对，键是'custom'，值是后面的 CustomStrategy。

#### 2. 初始化一个passport对象后，现在看`initialize`方法

```
Authenticator.prototype.initialize = function(options) {
  options = options || {};
  this._userProperty = options.userProperty || 'user';
  
  return this._framework.initialize(this, options);
};
```
这里调用了之前的`this._framework`中的`initialize`方法，这个方法定义是在`./middleware/initialize`

```
// ./middleware/initialize
module.exports = function initialize(passport) {
  
  return function initialize(req, res, next) {
    req._passport = {};
    req._passport.instance = passport;

    if (req.session && req.session[passport._key]) {
      // load data from existing session
      req._passport.session = req.session[passport._key];
    }

    next();
  };
};
```
所以说之前的`return this._framework.initialize(this, options);`将options变量传进去有啥用呢，之后的函数就一个参数啊~

这里是将之前的 Passport 实例 绑在 req上(在Koa框架中可以绑在 ctx上)，然后尝试取出请求中的session，应该是给默认的session策略用吧

#### 3. `authenticate`方法

添加完策略后，使用下面的代码，添加鉴权中间件
```
router.get('/', passport.authenticate('custom', {
  session: false
}), async (ctx, next) => {
  ctx.body = 'hello';
  await next();
});

// 或
app.use(passport.authenticate('custom'), { session: false });
```
清晰起见，把之前的策略定义也放在这里
```
new CustomStrategy((req, done) => {
  const { authorization } = req.headers;
  if (authorization === 'token') {
    return done(null, { user: 1 });
  } else {
    done(null, false);
  }
})
```

下面就是`authenticate`方法部分的代码了

```
Authenticator.prototype.authenticate = function(strategy, options, callback) {
  return this._framework.authenticate(this, strategy, options, callback);
};

// ./middleware/authenticate
module.exports = function authenticate(passport, name, options, callback) {
  if (typeof options == 'function') {
    callback = options;
    options = {};
  }
  options = options || {};
  
  var multi = true;
  
  // 支持多个策略
  if (!Array.isArray(name)) {
    name = [ name ];
    multi = false;
  }
  
  return function authenticate(req, res, next) {
    if (http.IncomingMessage.prototype.logIn
        && http.IncomingMessage.prototype.logIn !== IncomingMessageExt.logIn) {
      require('../framework/connect').__monkeypatchNode();
    }
    
    var failures = [];
    
    // 错误处理
    function allFailed() {
        ...
    }
    
    (function attempt(i) {
      var layer = name[i];
      // If no more strategies exist in the chain, authentication has failed.
      if (!layer) { return allFailed(); }
    
      // 根据name取出策略
      var prototype = passport._strategy(layer);
      if (!prototype) { return next(new Error('Unknown authentication strategy "' + layer + '"')); }
    
      // 创建一个策略实例
      var strategy = Object.create(prototype);
      
      strategy.success = function(user, info) {
        // 如果鉴权成功
      };
      
      strategy.fail = function(challenge, status) {
        // 如果鉴权失败
      };
      
      strategy.redirect = function(url, status) {
        // 重定向
      };
      

      strategy.pass = function() {
        next();
      };

      strategy.error = function(err) {
        // 出错
      };
      
      // ----- END STRATEGY AUGMENTATION -----
    
      strategy.authenticate(req, options);
    })(0); // attempt
  };
};
```
再看看`CustomStrategy`的定义

```
// passport-strategy
function Strategy() {
}

Strategy.prototype.authenticate = function(req, options) {
  throw new Error('Strategy#authenticate must be overridden by subclass');
};

module.exports = Strategy;

// passport-custom
function Strategy(verify) {
	if (!verify) {
		throw new TypeError('CustomStrategy requires a verify callback');
	}
	passport.Strategy.call(this);
	this.name = 'custom';
	this._verify = verify;
}

util.inherits(Strategy, passport.Strategy);

Strategy.prototype.authenticate = function (req) {
	var self = this;

	function verified(err, user, info) {
	    // 在这里调用了 authenticate 中给 strategy 定义的各种方法
		if (err) {
			return self.error(err);
		}
		if (!user) {
			return self.fail(info);
		}
		self.success(user, info);
	}

	try {
		this._verify(req, verified);
	} catch (ex) {
		return self.error(ex);
	}
};
```
这是一个典型的适配器模式。用户只要将自己的函数 fn 作为参数传入`new CustomStrategy(fn)`，这个函数就被包装成了 passport 可以 use 的策略。

当运行`passport.authenticate(name, options)`时，首先会根据name，取出对应的Strategy，然后调用 Strategy.authenticate()，这个函数会调用用户编写的策略，然后根据用户返回的信息，分别调用错误处理`Strategy.error()`或者鉴权失败`Strategy.fail()`或者鉴权成功`Strategy.success()`。

这里的options可以传入如下配置：
* failureFlash: 出错时会在flash的信息。//这个在koa中需要在这之前调用其他插件才能实现，express可以直接传递
* failureMessage: 将报错信息放在session中返回 //同上
* failureRedirect: 报错后将网址重定向到新的网址，这个可以直接使用
* failWithError: 如果鉴权出错，后台是否会报错
* successFlash: Koa不能直接使用
* successMessage: Koa不能直接使用
* successRedirect： 成功后将网址重定向到新的网址，这个可以直接使用
* successReturnToOrRedirect：不能直接使用
* assignProperty：鉴权成功后，用户的信息挂在req的哪个属性下。// 在koa-passport包中是挂在ctx.state的哪个属性下

### 三. 总结

上面主要讲了Passport.js的设计思路，它将每个策略抽象成了一个 Strategy 类，每个Strategy类只要实现了authenticate方法，并在内部调用用户的具体实现，并针对成功失败出错三种情况调用success fail error方法，那么这个Strategy就能被用户使用。

Passport.js 强大的地方还在于它有一个策略库，可以在里面搜到比较知名网站的许多鉴权策略，下载安装加一些配置就可以用了（遗憾的是高质量的策略都是国外公司滴）。

最后不得不吐槽的就是这个包对 koa 的支持，[作者 13 年就说会考虑对Koa的官方支持](https://github.com/jaredhanson/passport/issues/194)，结果18年还是没信儿。但是的确这个包如果想直接支持koa很难，因为在authenticate中req res 和koa的 ctx 方法区别还是不少的。

最后我发现通过`Authenticator.prototype.framework`方法，可以做到更换底层的 `initialize`和`authenticate`，所以我写了[`passport-koa`](https://www.npmjs.com/package/passport-koa)模块来支持 Koa，相比之下[`koa-passport`](https://www.npmjs.com/package/koa-passport)包是对原始的 passport.js 做了一个 wrapper，这个不光重写了那两个方法，还要托管别的方法。我觉得插件这种解决方案更优雅，欢迎大家使用。

