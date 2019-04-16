---
layout: simple-article
title: ES6 之 Proxy代理
categories:
    - JavaScript
    - ES6
tags:
    - ES6
---
### 前言
Proxy 是 ES6 中新增的一个元编程特性。

Proxy 代理是一个特殊的对象，它封装了另外一个普通对象。在代理上执行各种操作时，不仅会将操作转发给被代理的对象，还有机会执行额外的逻辑。
<!-- more -->
> 元编程是指把程序的逻辑转向关注自身(或自身的运行环境)，要么是为了查看自己的结构，要么是为了修改它。元编程的主要价值是拓展语言的一般机制来提供额外的功能。元编程是对程序本身行为特性的编程。比如 a.isPrototypeOf(b) 查看 a 是否是 b 的原型，这就是一种元编程，称为内省。


### 一. 语法

#### 1.1 基本用法

```
// 目标对象
const target = {
    val: 1,
};

// 当执行一个操作时定义代理行为的函数
const handler = {
    // target 就是目标对象, target === target
    // context 是生成的 Proxy 对象, context === proxyObj
    // 这里的 get 是一个 trap
    get(target, key, context) {
        if (Reflect.has(target, key)) {
            return Reflect.get(target, key);
        } else {
            throw 'No such key';
        }
    },
    set(target, key, val, context) {
        if (Reflect.has(target, key)) {
            return Reflect.set(target, key, val);
        } else {
            throw 'No such key';
        }
    }
}

const proxyObj = new Proxy(target, handler);

proxyObj.val; // 1
proxyObj.val = 2; // 2

proxyObj.val1; // No such key
proxyObj.val1 = 2; // No such key
```
创建一个代理需要一个目标对象 target 和 包含 trap 函数 的对象 handler。代理通过 trap 函数拦截对目标对象的操作，加入代理处理的逻辑。

#### 1.2 Handler 拥有的 trap 函数类型 与 Reflect 对象

trap 函数以及对应的触发时机主要有以下几种：
| trap 函数 | 触发时机 | 触发方式 | 
| --- | --- | --- |
|`get`| 访问代理的属性时会被触发|`Reflect.get(proxyObj, key)`/属性运算符
|`set`| 设置代理的属性时会被触发|`Reflect.set(proxyObj, key, val`)/赋值运算符
|`apply`|**当目标对象是函数**，将代理作为普通函数或者方法调用时会被触发|`Reflect.apply(proxyObj, thisObj, [...args])` 将thisObj 作为this 调用 proxyObj([...args])/ call(...)/ apply(...) / (...)
|`constructor`|**当目标对象是函数**，将代理作为构造函数调用|`Reflect.constructor(proxyObj, [...args])` / `new proxyObj([...args])`
|`deleteProperty`|在代理上删除一个属性|`Reflect.deleteProperty(proxyObj, key);` / `delete` 操作符 
|`defineProperty`|在代理上设置一个属性描述符|`Object/Reflect.defineProperty(proxyObj, key, descriptor)`
|`getOwnPropertyDescriptor`|从代理提取一个属性描述符|`Object/Reflect.getOwnPropertyDescriptor(proxyObj, key)`
| `getPrototypeOf`| 得到代理的 [[Prototype]] | `Object/Reflect.getPrototypeOf(proxyObj)`
| `setPrototypeOf` | 设置代理的[[Prototype]] | `Object/Reflect.setPrototypeOf(proxyObj, otherObj)`
| `has` | 检查代理是否拥有或者原型链上是否有某属性 | `Reflect.has(proxyObj, key)`/ `Object#hasOwnProperty(key)`/ `'key' in proxyObj`
| `ownKeys` | 提取代理自己的属性或者符号属性 | `Reflect.ownKeys(proxyObj) // 既有 属性也有符号属性`/ `Object.keys(proxyObj)`/`Object.getOwnPropertyNames(proxyObj)`/`Object.getOwnSymbolProperties(proxyObj)`/`JSON.stringify(proxyObj)`
| `preventExtensions` | 使代理不可扩展 | `Reflect.preventExtensions(proxyObj)` / `Object.preventExtensions(obj)`
| `isExtensible` | 检查代理是否可扩展 | `Reflect.isExtensible(proxyObj)`/ `Object.isExtensible(obj)`

当对代理对象执行一些操作时，如果符合上述的时机，那就会执行 handler 中对应的 trap 函数。也就实现了对对象操作的拦截。

Reflect 是一个内置的对象，它提供拦截 JavaScript 操作的方法。

由上可见，handler 拥有的 trap 类型是与 Reflect 拥有的方法是一一对应的。实际上就是这么设计的。

#### 1.3 取消代理

```
let obj = {
    a: 1
};

let handlers = {
    get(target, key, context) {
        return Reflect.get(target, key, context)
    }
}

let { proxy: pobj, revoke: prevoke } = Proxy.revocable(obj, handlers);

// 1
console.log(pobj.a);

prevoke();

// TypeError: Cannot perform 'get' on a proxy that has been revoked
console.log(pobj.a);
```

可见使用 `Proxy.revocable` 不止会返回一个代理对象，还会返回一个 revoke 函数，当调用 revoke 函数后，代理对象会被取消，就无法使用了。

#### 1.4 如何判断一个对象是否是 Proxy

使用 instaceof 来判断是行不通的，比如用 `proxyObj instanceof Proxy` 是 false。 

https://stackoverflow.com/questions/36372611/how-to-test-if-an-object-is-a-proxy

参考上述链接。要么在代理的 get trap 中增加返回 `__isProxy`标志位为 true 的逻辑；要么使用 node v10 中新增的 `util.types.isProxy` 

```
...
     get: (target, key) => {
        if (key !== "__isProxy") {
          return target[key];
        }

        return true;
      }
...
```

### 二. 应用

#### 2.1 从对象中抽离校验逻辑

```
function createValidator(target, validator) {
    return new Proxy(target, {
        _validator: validator,
        set(target, key, value, proxy) {
            if (target.hasOwnProperty(key)) {
                let validator = this._validator[key];
                if (!!validator(value)) {
                    return Reflect.set(target, key, value, proxy);
                } else {
                    throw Error(`Cannot set ${key} to ${value}. Invalid.`);
                }
            } else {
                throw Error(`${key} is not a valid property`)
            }
        }
    });
}

const personValidators = {
    name(val) {
        return typeof val === 'string';
    },
    age(val) {
        return typeof val === 'number' && val > 18;
    }
}

class Person {
    constructor(name, age) {
        this.name = name;
        this.age = age;
        return createValidator(this, personValidators);
    }
}

const bill = new Person('Bill', 25);

bill.name = 'Bill Gates'
bill.age = 52;
console.log(bill); // Person { name: 'Bill Gates', age: 52 }

// Error 
bill.name = 0;
bill.age = 'Bill';
bill.age = 15;
```

#### 2.2 私有属性

JS 目前是没私有属性的。但是可以通过 Proxy 实现这个逻辑

具体来说就是在handler 的 get set trap 中对 key 进行判断，如果是私有属性，就抛出错误。

```
let api = {
    _apiKey: '123abc456def',
    getUsers: function () { },
    getUser: function (userId) { },
    setUser: function (userId, config) { }
};

const RESTRICTED = ['_apiKey'];

api = new Proxy(api, {
    get(target, key, proxy) {
        if (RESTRICTED.indexOf(key) > -1) {
            throw Error(`${key} is restricted. Please see api documentation for further info.`);
        }
        return Reflect.get(target, key, proxy);
    },
    set(target, key, value, proxy) {
        if (RESTRICTED.indexOf(key) > -1) {
            throw Error(`${key} is restricted. Please see api documentation for further info.`);
        }
        return Reflect.get(target, key, value, proxy);
    }
});
// 以下操作都会抛出错误 
console.log(api._apiKey);
api._apiKey = '987654321';
```

#### 2.3 预警和拦截

不想让其他开发者删除 noDelete 属性，还想让调用 oldMethod 的开发者了解到这个方法已经被废弃了，或者告诉开发者不要修改 doNotChange 属性，那么就可以使用 Proxy 来实现

```
let dataStore = {
    noDelete: 1235,
    oldMethod: function () {/*...*/ },
    doNotChange: "tried and true"
};

const NODELETE = ['noDelete'];
const NOCHANGE = ['doNotChange'];
const DEPRECATED = ['oldMethod'];

dataStore = new Proxy(dataStore, {
    set(target, key, value, proxy) {
        if (NOCHANGE.includes(key)) {
            throw Error(`Error! ${key} is immutable.`);
        }
        return Reflect.set(target, key, value, proxy);
    },
    deleteProperty(target, key) {
        if (NODELETE.includes(key)) {
            throw Error(`Error! ${key} cannot be deleted.`);
        }
        return Reflect.deleteProperty(target, key);
    },
    get(target, key, proxy) {
        if (DEPRECATED.includes(key)) {
            console.warn(`Warning! ${key} is deprecated.`);
        }
        var val = target[key];
        return typeof val === 'function' ? function (...args) {
            Reflect.apply(target[key], target, args);
        } : val;
    }
});

// these will throw errors or log warnings, respectively 
dataStore.doNotChange = "foo";
delete dataStore.noDelete;
dataStore.oldMethod();
```

#### 2.4 多继承

```
Object.setPrototypesOf = function (target, ...prototypes) {
    if (prototypes.length < 1) {
        return target;
    }
    const handler = {
        get(target, key, context) {
            if (Reflect.has(target, key, context)) {
                return Reflect.get(target, key, context);
            } else {
                for (const P of target[Symbol.for('[[Prototype]]')]) {
                    if (Reflect.has(P, key)) {
                        return Reflect.get(P, key, context)
                    }
                }
            }
        }
    }

    const pobj = new Proxy(target, handler);

    pobj[Symbol.for('[[Prototype]]')] = prototypes;

    return pobj;
}

const obj1 = { name: 'obj1', foo() { console.log('obj1.foo: ', this.name) } }
const obj2 = { name: 'obj2', bar() { console.log('obj2.bar: ', this.name) } }
const obj4 = Object.setPrototypesOf({ name: 'obj4', baz() { this.foo(); this.bar() } }, obj1, obj2)

obj4.baz()
```
`Symbol.for('[[Prototype]]') `只是一个符合语意方便引用的值，没什么特别的。这里它是一个命名钩子


### 引用

[实例解析ES6 Proxy使用场景](http://www.w3cplus.com/javascript/use-cases-for-es6-proxies.html)

[你不知道的JavaScript](https://book.douban.com/subject/27620408/)