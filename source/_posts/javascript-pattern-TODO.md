---
title: JavaScript 模式
categories:
    - JavaScript
tags:
    - JavaScript
---

# 一. 基本技巧

### 1.1 单一var模式

**只使用一个 var(let) 在函数的顶部进行变量声明以及初始化**

优点：
- 方便找到函数需要的所有局部变量
- 防止变量在定义前被使用
- 编码更少
```js
function func() {
  let a = 1,
      b = 2,
      sum = a + b,
      obj = {};
  // function body
}
```

### 1.2 for 循环不要重复获取长度
**如果要缓存的数组是HTMLCollection，涉及DOM操作，那么缓存遍历的数组的长度**

```js
for (let i = 0, length = arr.length; i < length; i++) {
  // 操作
  // 如果中间数组长度更改了，那么需要修改 length
}
```


for 循环在对类数组对象进行遍历，尤其是对 HTMLCollection 对象进行遍历，如果每次都要获取长度，那么是非常耗性能的
```js
// 假设 arr 是一个 HTMLCollection
for (let i = 0; i < arr.length; i++) {
  // 操作
}
```

### 1.3 数值解析与转换

转换指确定字符串只包含数字，并且数字的基数是 10。解析指从字符串中解析到开头的数字部分，后面可能有别的字符，并且可以指定基数。
- **将一个字符串解析为数字时，使用parseInt，为了代码清晰要传递第二个参数来明确指定基数**
- **将一个字符串转换为数字时，使用 "+" 或者 Number() 来转换，这样速度更快**

### 1.4 抛出自定义错误
throw 除了可以抛出 new Error 产生的对象，也可以抛出任意对象，可以利用这个特点在某些时候优雅的处理错误
```js
try {
  throw {
    name: "MyError",
    message: "balabala",
    process: errorHandler,
  }
} catch (e) {
  console.log(e.message)
  e.errorHandler()
}
```
# 二. 函数
### 2.1 立即执行函数
通过立即执行函数可以让函数立即执行，并确保变量不会污染全局变量，尽管现在通过 let const 可以获得块级作用域，但是立即执行函数还是非常常用
```js
let result = (function (a, b) {
  // 操作
})(bar, foo)
```

### 2.2 初始化分支
**在知道某个条件在整个程序周期内都不会发生改变的时候，仅对该条件只测试一次**
```js
let utils = {
  addListener: function (el, func, type) {
    if (typeof el.addEventListener === 'function') {
      el.addEventListener(type, func, false);
    } else if (el.attachEvent === 'function') {
      el.attachEvent('on' + type, fn);
    } else {
      el['on' + type] = fn;
    }
  },
  removeListener: function (el, func, type) {
    // ...
  }
}

// 修改为
let utils = {
  addListener: null,
  removeListener: null,
}
if (typeof window.addEventListener === 'function') {
  utils.addListener = function (el, func) {
    el.addEventListener(type, func, false);
  }
} else if (window.attachEvent === 'function') {
  utils.addListener = function (el, func) {
    el.attachEvent('on' + type, fn);
  }
} else {
  utils.addListener = function (el, func) {
    el['on' + type] = fn;
  }
}
```

# 三. 对象
### 3.1 实现对象的私有成员
通过闭包实现对象的私有成员。
```js
function Foo {
  // 私有属性
  let name = 'lwt';
  this.getName = function() {
    return name;
  }
  this.setName = function (val) {
    name = val;
  }

  // 私有方法
  let privateMethod = function() {
    // do something
  }
  this.doSomething = function() {
    // ....
    privateMethod();
  }
}

// prototype 中实现私有属性，创建的对象共享这个属性
Foo.prototype = (function() {
  let browser = 'chrome';
  return {
    getBrowser() {
      return browser;
    }
  }
})()
```
### 3.2 私有静态成员
```js
let Gadget = (function () {
  let counter = 0,
      NewGadget;
  // 新的构造函数
  NewGadget = function () {
    counter += 1;
  }
  NewGadget.prototype.getLastId = function () {
    return counter;
  }
  return NewGadget;
})()

// 所有构造出来的对象共享 couner 这个静态属性，并且不能直接访问这个属性

// 使用
let a = new Gadget();
// 1
console.log(a.getLastId())
let b = new Gadget();
// 2
console.log(a.getLastId())
```

### 3.3 模块模式

#### 1. 对象形式
```js
utils.array = (function (_) {
  // 依赖
  // 这里只是示范一下
  let { map } = _,
  // 私有属性
      array = "[object Array]",
  // 私有方法
      ops = Object.prototype.toString;

  // 这里进行一些一次性的初始化过程
  // ......
  // 共有API
  return {
    isArray(a) {
      return ops.call(a) === array;
    }
  }
})(lodash)

// 使用方法
utils.array.isArray([1, 2, 3]);
```

#### 2. 构造函数形式
```js
utils.Array = (function(_) {
  // 依赖
  let { map } = _,
  // 私有属性
      array = "[object Array]",
  // 私有方法
      ops = Object.prototype.toString;
  // 初始化过程
  // ......
  // 构造函数
  Constr = function (o) {
    this.elements = this.toArray(o);
  }
  // 共有API
  Constr.prototype = {
    constructor: utils.Array,
    toArray(o) {
      return Array.from(o);
    }
  }
  // 返回构造函数
  return Constr;
})(lodash)

// 使用方法
let arr = new utils.Array(obj);
```

**向模块模式中传递依赖或者参数可以通过立即执行函数的参数来传递**

