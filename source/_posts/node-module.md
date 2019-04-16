---
title: 模块加载原理与 module 源码阅读
date: 2018-11-17 20:12:37
categories:
    - 服务器端
    - Node.js
tags:
    - Node.js

---



- 核心模块：
    - C++ 核心模块
    - Node.js 内置模块
- 文件模块：
    - 用户源码模块
    - C++ 扩展

C\+\+ 核心模块与C++ 扩展的主要区别在于核心模块在 Node 源码中，并且编译进 Node.js 的可执行二进制文件中；C++扩展以动态链接库的形式存在
<!-- more -->
### 一. Node.js 模块类型
#### 1. C++核心模块
- http-parser 解析http报文 
- openSSL 安全套接字层协议库
- zlib 数据压缩库
- 等等

#### 2. Node.js 内置模块
lib 文件夹下的 js 文件就是一个个的 Node 内置模块。

#### 3. 用户源码模块
用户源码模块就是用户在Node项目中的JS源码和第三方包的模块。这些模块是在程序运行时，在需要时被 require 函数加载的。

**平时在源码中使用的 require() 其实就是 lib/module.js 中的 Module 类实例对象的 require()**

> 一个 Module类的实例对象就是一个用户源码模块，用户通过 require() 引入的文件代码与其在 vm 沙盒中的结果就是这个模块的核心。

```
// lib/module.js
Module.prototype.require = function(path) {
  assert(path, 'missing path');
  assert(typeof path === 'string', 'path must be a string');
  return Module._load(path, this, /* isMain */ false);
};
```
require 函数调用了 _load()，并声明 isMain 为 false(不是入口模块)
```
// lib/module.js
Module._load = function(request, parent, isMain) {
  ...
  
  var filename = Module._resolveFilename(request, parent, isMain);
  var cachedModule = Module._cache[filename];
  
  // 1. 如果 module 已经在缓存中了，直接返回它的 exports 对象
  if (cachedModule) {
    return cachedModule.exports;
  }
  
  // 2. 如果是原生模块，使用 NativeModule.require，并返回结果
  if (NativeModule.nonInternalExists(filename)) {
    debug('load native module %s', request);
    return NativeModule.require(filename);
  }
  
  // 3. 创建一个module对象，并存入缓存
  var module = new Module(filename, parent);
  
  // 4. 如果是入口模块，进行标记
  if (isMain) {
    process.mainModule = module;
    module.id = '.';
  }
  Module._cache[filename] = module;
  
  // 5. 加载模块
  tryModuleLoad(module, filename);
  
  return module.exports;
}
```
以上代码大致是注释的五步，第五步调用了 tryModuleLoad 函数来加载模块。
```
function tryModuleLoad(module, filename) {
  var threw = true;
  try {
    // 有错就抛出错误
    module.load(filename);
    threw = false;
  } finally {
    // 如果有错那就从缓存中去除错误module对象
    if (threw) {
      delete Module._cache[filename];
    }
  }
}
```
tryModuleLoad 调用了 load()
```
Module.prototype.load = function(filename) {
  this.filename = filename;
  this.paths = Module._nodeModulePaths(path.dirname(filename));

  var extension = path.extname(filename) || '.js';
  if (!Module._extensions[extension]) extension = '.js';
  Module._extensions[extension](this, filename);
  this.loaded = true;
};
```
load 函数会根据不同的文件后缀名使用不同的载入规则。
- js: Module._extensions['.js']
- json: Module._extensions['.json']
- node: Module._extensions['.node']

这里看用户模块，所以主要看下 js 的载入规则
```
Module._extensions['.js'] = function(module, filename) {
  
  // 1. 同步读取源码的内容
  var content = fs.readFileSync(filename, 'utf8');
  
  // 2. 调用 module._compile() 编译源码
  module._compile(internalModule.stripBOM(content), filename);
};
```
调用了 module._compile() 来编译代码

```
Module.wrap = NativeModule.wrap;
...
Module.prototype._compile = function(content, filename) {
    ...
    // 1. 创建闭包源码
    var wrapper = Module.wrap(content);
    
    // 2. 用 vm 编译 wrapper
    var compiledWrapper = vm.runInThisContext(wrapper, {
        filename: filename,
        lineOffset: 0,
        displayErrors: true
    });
    ...
    var dirname = path.dirname(filename);
    var require = internalModule.makeRequireFunction.call(this);
    var args = [this.exports, require, this, filename, dirname];
    var depth = internalModule.requireDepth;
    if (depth === 0) stat.cache = new Map();
    // 3. 传入参数执行 vm 
    var result = compiledWrapper.apply(this.exports, args);
    if (depth === 0) stat.cache = null;
    return result;
}
```
一个模块经过闭包化之后，就变成了接受 exports、require、module、\_\_filename、__dirname的闭包函数，类似
```
(function(exports, require, module, __filename, __dirname){
    // 用户模块源码
})
```
这就是为什么可以在代码中直接使用 exports module require 的原因。

这个闭包只在第一次加载模块时执行了一次，之后就一直存在在模块缓存中，只要不手动清除缓存，就不会重新加载这个模块。所以模块逻辑代码只会被执行一次。

注意，五个参数中的 module 就是 Module 类的对象实例，所以对 module.exports 赋值实际上就是对传入的 module 对象进行赋值。参数中的 require 函数就是包装后的 Module.prototype.require

##### 用户源码模块载入流程：
1. 开发者在代码中调用 require()，这相当于调用 Module._load
2. 如果有缓存，直接返回 module.exports
3. 如果没有缓存，闭包化对应文件的源码，传入 exports、require、module、\_\_filename、__dirname 参数执行
4. 对应文件中代码中往往会对 module.exports 或者 exports 进行赋值
5. Module._load 返回新建 module 的 exports 给上游

##### 入口模块
入口模块指在命令行执行 node <filename> 时，指定的文件。

入口模块调用方式： `Module._load(process.argv[1], null, true)`

相对于其他模块的require函数，第一个参数变成了处理过后的命令行参数(转化为了绝对路径)；第二个 parent 参数变为 null，因为入口模块没有父模块；第三个参数变为 true，生成的 module 会赋值给 process.mainModule

#### 4. C++扩展

用户模块(.js)和C++扩展模块(.node)加载时的区别在于 Module.prototype.load  函数会根据不同的文件后缀名使用不同的载入规则

现在看一下`Module._extensions['node']`
```
Module._extensions['.node'] = function(module, filename) {
  // 加载 .node 模块
  return process.dlopen(module, path._makeLong(filename));
};
```

#### 总结
- C++ 核心模块会通过 `NODE_MODULE_CONTEXT_AWARE_BUILTIN`等宏将不同模块注册进 Node.js C++ 核心模块链表中
- Node.js 内置模块会在Node 编译时被写入 C++ 源码中并被编译到 Node 的可执行二进制文件中，并在合适时机(被 requrire 时)闭包化导出
- 用户源码模块会在首次执行 require 时被读取源码并闭包化导出，然后再加入模块缓存中
- C++ 拓展会在首次执行 require 时通过 uv_dlopen 加载该扩展.node 文件(动态链接库文件)，在链接库内部把模块注册函数赋值给 modepending，然后将执行 require 时传入的 module 和 exports 两个对象传入模块注册函数进行导出。

这四个模块加载原理我目前就 用户源码模块 的加载原理是理解最清楚的，其他模块的加载过程都涉及了C\++代码，我不是很会 C++，所以只有一个感性的认识。抽空还是应该好好学学C++啊！

### 参考
- [来一打Node.js C++ 扩展](https://book.douban.com/subject/30247892/)
- [详解 Node.js vm模块](http://www.alloyteam.com/2015/04/xiang-jie-nodejs-di-vm-mo-kuai/)
