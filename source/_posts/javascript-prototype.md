---
layout: simple-article
title: 理解原型链
date: 2018-01-30 20:11:20
categories:
    - JavaScript
tags:
    - JavaScript
---


### 一. 原型链

#### 1. 是什么
JS 中每个对象都有一个 [[Prototype]] 的内置属性，这个属性就是对于其他对象的引用。

> **除了使用 `Object.create(null)`产生的对象，其他所有的对象创建后，[[Prototype]]属性都会是个非空的值**

当引用一个对象的属性时，会触发 [[GET]] 操作。

对默认的 [[GET]] 操作：
1. 检查对象本身是否有这个属性
2. 如果对象本身没有这个属性，检查对象的原型中是否有这个属性
3. 如果对象的原型中也没有这个属性，那就继续查找对象原型的原型，也就是原型链。

#### 2. 遍历原型链/查找原型链
使用 for... in 可以遍历原型链，任何在原型链中且为 enumerable 的属性都会被枚举。

使用 in 操作符来检查属性在对象中是否存在，也会查找原型链。

#### 3. 原型链的尽头 Object.prototype
所有普通原型链的尽头都会指向内置的 Object.prototype，这个对象有许多JS中通用的功能，每个JS对象都可以访问它的方法。

`Object.create(null)` 产生的对象是个例外。

以下方法，所有普通对象都可以调用：
| 方法名 | 用法 | 说明
| --- | --- | --- |
| `hasOwnProperty` | `obj.hasOwnProperty(key)` |用来检测一个对象是否含有特定的自身属性；和 in 运算符不同，该方法会忽略掉那些从原型链上继承到的属性。
| `isPrototypeOf` | `myObj.isPrototypeOf(obj)`| myObj 是否在 obj 的原型链上
| `propertyIsEnumerable` | `obj.propertyIsEnumerable(key)` | 验证自身的属性是否是可枚举的，如果是原型链上的属性，会返回 false
| `toString` 
| `valueOf`

#### 4. 函数实例的原型对象
如果使用 new 关键字构造的对象，那么这个对象的原型对象指向构造函数的原型对象。
```
function A() {
  this.key = 'key';
  this.value = 'value';
}

let a = new A();

a.__proto__ === A.prototype
```

#### 5. 例子
![data:image](https://camo.githubusercontent.com/71cab2efcf6fb8401a2f0ef49443dd94bffc1373/68747470733a2f2f757365722d676f6c642d63646e2e786974752e696f2f323031382f332f31332f313632316538613962636230383732643f773d34383826683d35393026663d706e6726733d313531373232)

对象的 \__proto__ 最终都会指向 Object.prototype，而 Object.prototype.\__proto__ === null:
1. Foo.prototype 是一个对象，是 Object 的实例，所以 Foo.prototype.\__proto__ === Object.prototype
2. Function.prototype 是一个对象，是 Object 的实例，所以 Function.prototype.\__proto__ === Object.prototype

函数是 Function 的实例：
1. Foo 是一个函数，也就是 Function 的实例，所以 Foo.\__proto__ === Function.prototype
2. Function 本身也是 Function 的实例，所以 Function.\__proto__ === Function.prototype
3. Object 也是一个函数，Object.\__proto__ === Function.prototype

原型对象的 contructor 如果不改变的话，会指向构造函数
1. Foo.prototype.constructor === Foo
2. Function.prototype.constructor === Function 


### 二. Object中相关方法

Object 对象有许多方法与原型链有关。这里总结一下。

| 方法 | 用法 | 说明
| --- | --- | ---
| `Object.create` | `son = Object.create(parent)` | 创建一个新对象，使用现有的对象来作为新创建的对象的__proto__
| `Object.getPrototypeOf` | `Object.getPrototypeOf(son) === parent` | 返回指定对象的原型对象，相当于 `son.__proto__ === parent`
| `Object.setPrototypeOf` | `Object.setPrototypeOf(son, parent)` | 设置一个对象的原型，相当于 `son.__proto__ = parent`


> 由于现代 JavaScript 引擎优化属性访问所带来的特性的关系，更改对象的 [[Prototype]]在各个浏览器和 JavaScript 引擎上都是一个很慢的操作。其在更改继承的性能上的影响是微妙而又广泛的，这不仅仅限于 obj.__proto__ = ... 语句上的时间花费，而且可能会延伸到任何代码，那些可以访问任何[[Prototype]]已被更改的对象的代码。如果你关心性能，你应该避免设置一个对象的 [[Prototype]]。相反，**你应该使用 Object.create()来创建带有你想要的[[Prototype]]的新对象**。

### 三. 实例

#### 1. 使用 Object.create 关联两个对象

```
let a = {
    b: 1
}

let c = Object.create(a);

c.__proto__ === a;  // true
Object.getPrototypeOf(c) === a // true
```



### 参考

[Object MDN 文档](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Object)