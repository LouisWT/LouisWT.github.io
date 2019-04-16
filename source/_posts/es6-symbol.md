---
layout: simple-article
title: ES6 之 Symbol
categories:
    - JavaScript
    - ES6
tags:
    - ES6
---
Symbol 是 ES6 新引入的数据类型，是除了 Number String Boolean Object undefined null 以外的第七个数据类型。
<!-- more -->
### 一. 用法

#### 1.1 创建一个唯一标识符 

Symbol 可以用来创建一个独一无二的值。

```
let s1 = Symbol('foo')
let s2 = Symbol('foo');

s1 === s2 // false
```
可见即使是相同字符串创建出来的 Symbol 值也是不同的。

#### 1.2 作为对象的键值

```
const s = Symbol('key')
let a = {
    [s]: 1,
}

a[s] // 1
```

不能用 . 运算符的写法，因为它不会计算属性名，跟的总是字符串

#### 1.3 获取对象所有的 Symbol 属性

使用 Symbol 的属性名，不会出现在 for ... in, for ... of 循环中，可以使用 `Object.getOwnPropertySymbols` 来获取一个对象的符号属性名。如果既想获取普通属性也想获取符号属性，可以用 `ReFlect.ownKeys()`

```
let a = {
    [Symbol()]: 1,
    b: 2,
}

Object.getOwnPropertySymbols(a) // [Symbol()]
Reflect.ownKeys(a) // ["b", Symbol()] 
```

#### 1.4 注册全局符号
由于每个符号是唯一的，那么在使用时不免将符号变量传来传去的来使用。这时候就可以使用注册全局符号的方式来避免。

```
let b = Symbol('key3')
let a = {
    [Symbol.for('key1')]: 1,
    [Symbol('key2')]: 2,
    [b]: 3,
}

a[Symbol.for('key1')] // 1
a[Symbol('key1')] // undefined
a[b] // 3
```
`Symbol.for(key)` 使用给定的key在全局Symbol中搜索现有的symbol，如果找到则返回该symbol。否则将使用给定的key在全局symbol注册表中创建一个新的symbol。

`Symbol.keyFor(symbol)` 与 `Symbol.for(key)`的过程相反，它是根据一个 symbol 值在全局注册表中搜寻相应的键值

### 二. 公开符号

ES6 预先定义了一些内置符号。这些符号主要是为了提供一些专门的元属性。

#### 2.1 `Symbol.iterator`
Symbol.iterator 表示任意对象的一个专门属性，JS会自动在这个属性上寻找一个方法，用这个方法构造一个迭代器来消耗这个对象的值。

可以通过修改这个属性值来定制迭代器逻辑。

```
const arr = [1,2,3,4,5,6,7];

for (let v of arr) {
    console.log(v) // 1 2 3 4 5 6 7
}

arr[Symbol.iterator] = function*() {
    for (let idx = 1; idx < this.length; idx += 2) {
        yield this[idx];
    }
}

for (let v of arr) {
    console.log(v) // 2 4 6
}
```

#### 2.2 `Symbol.toStringTag` 与 `Symbol.hasInstance`
`Symbol.toStringTag` 影响的是对象的 toString() 的结果

`Symbol.hasInstance` 影响的是 instaceof 的结果

```
function Foo(greeting) {
    this.greeting = greeting;
}

Foo.prototype[Symbol.toStringTag] = "Foo";

Object.defineProperty(Foo, Symbol.hasInstance, {
    value: function(instance) {
        return instance.greeting === 'hello';
    }
})

let a = new Foo('hello');
let b = new Foo('world');

b[Symbol.toStringTag] = 'Bar'

a.toString() // [object Foo]
String(b) // [object Bar]

a instanceof Foo // true
b instanceof Foo // false
```

Symbol.hasInstance 属性默认 writable: false，所以要用 Object.defineProperty 方式来定义。

#### 2.3 `Symbol.isConcatSpreadable`
用于配置某对象作为 Array.prototype.concat 方法的参数时是否展开其数组元素。

1. 对数组来说
```
let a = [4,5,6];
let b = [1,2,3];

a.concat(b); // [4,5,6,1,2,3]

b[Symbol.isConcatSpreadable] = false;
a.concat(b); // [4,5,6,[1,2,3]]
```
2. 类数组对象
```
let a = [1,2,3];
let arraylike = {
    [Symbol.isConcatSpreadable]: true,
    length: 2,
    0: 'hello',
    '1': 'world'
};

a.concat(arraylike) // [1, 2, 3, "hello", "world"]
```
将类数组对象的 Symbol.isConcatSpreadable 设置为 true，并指定 length 属性。就可以对这个类数组对象展开了。

#### 2.4 `Symbol.unscopables`

指用于指定对象值，其对象自身和继承的从关联对象的 with 环境绑定中排除的属性名称。

不建议使用 with 语法，所以这个符号最好也少用

#### 2.5 `Symbol.species`

`Symbol.species`的值会被构造函数用来创建派生对象

```
class MyArray extends Array {
  // 覆盖 species 到父级的 Array 构造函数上
  static get [Symbol.species]() { return Array; }
}
let a = new MyArray(1,2,3);

console.log(a instanceof MyArray); // true
console.log(a instanceof Array); // true

let mapped = a.map(x => x * x);

console.log(mapped instanceof MyArray); // false
console.log(mapped instanceof Array); // true
```

```
class MyArray extends Array {}
let a = new MyArray(1,2,3);

console.log(a instanceof MyArray); // true
console.log(a instanceof Array); // true

let mapped = a.map(x => x * x);

console.log(mapped instanceof MyArray); // true
console.log(mapped instanceof Array); // true
```
在实际应用场景中，如果你是一个库作者，在一个库的内部使用了一些内部类，但是在导出方法时想返回常规类型来供其他人使用，那么就可以覆写 Symbol.species。

[具体戳这里](https://stackoverflow.com/questions/38082965/symbol-species-example-from-mdn-not-making-sense)

#### 2.6 `Symbol.toPrimitive`
通过这个属性可以将对象转换为原始值

```
const obj1 = {};
console.log(+obj1);     // NaN
console.log(`${obj1}`); // "[object Object]"
console.log(obj1 + ""); // "[object Object]"

// 拥有 Symbol.toPrimitive 属性的对象
const obj2 = {
  [Symbol.toPrimitive](hint) {
    if (hint == "number") {
      return 10;
    }
    if (hint == "string") {
      return "hello";
    }
    return true;
  }
};
console.log(+obj2);     // 10      -- hint is "number"
console.log(`${obj2}`); // "hello" -- hint is "string"
console.log(obj2 + ""); // "true"  -- hint is "default"
```
hint 有三个值，`number/string/default`

#### 2.7 正则表达式 Symbol

- Symbol.match
- Symbol.split
- Symbol.replace
- Symbol.search

1. `Symbol.match`

为 true 表明这个对象是正则表达式；为false表示这个对象当做字符串使用。

```
'/bar/'.startsWith(/bar/) // TypeError: First argument to String.prototype.startsWith must not be a regular expression

const re = /foo/;
re[Symbol.match] = false;
'/foo/'.startsWith(re); // true
'/bar/'.startsWith(re); // false
```

2. `Symbol.split`
指向一个正则表达式的索引出分割字符串的方法

```
const exp = {
    pattern: 'in',
    [Symbol.split](str) {
        return str.split(this.pattern);
    }
}

'outinoutdayinday'.split(exp); // ["out", "outday", "day"]
```

这样感觉没啥意义，因为直接用 `'outinoutdayinday'.split('in');`也没啥区别

3. `Symbol.replace`/`Symbol.search`

一个替换匹配字符串的子串的方法.

一个返回一个字符串中与正则表达式相匹配的索引的方法

### 三. 总结
Symbol 单独用感觉主要是用来消除魔法字符串的，也就是说项目中那些约定的字符串，但是在 Symbol.for(key) 中使用的 key 其实还是另外一种形式的魔法字符串，只能说这好像并不能完全避免魔法字符串的使用。

公开符号主要是用于元编程的。

### 参考
- [ES6入门](http://es6.ruanyifeng.com/#docs/symbol)
- [MDN文档](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Symbol)