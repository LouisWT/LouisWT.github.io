---
layout: simple-article
title: this 解析
date: 2018-01-31 21:19:37
categories:
    - JavaScript
tags:
    - JavaScript
---
this 是在函数运行时被绑定的对象，只取决于函数的调用方式。
<!-- more -->

### 一. this 是什么
JS 中，this 关键字被自动定义在所有函数的作用域中。

this 是在运行是进行绑定的。**this 的绑定和函数声明的位置没有任何关系，只取决于函数的调用方式**。

### 二.判断 this

越前面优先级越高

##### 1. 函数是否在 new 中调用，如果是的话，函数的 this 绑定的是新创建的对象
```
const bar = new foo()
```

##### 2. 函数是否通过 call, apply, bind 显式绑定，如果是的话，this 绑定的是指定的对象
```
let bar = foo.call(obj2)

// bind 会返回由指定的this值和初始化参数改造的原函数拷贝
const _foo = foo.bind(obj2)
bar = _foo();
```

##### 3. 函数是否在某个上下文对象中调用，如果是的话，this 绑定的是那个上下文对象
```
const bar = obj1.foo()
```
##### 4. 如果都不是，使用默认绑定，严格模式下，绑定到 undefined，否则绑定到全局对象
```
const bar = foo()
```

### 三. 特殊绑定行为

#### 3.1 向 bind apply call 中传入 null 或 undefined 
这种情况下会使用默认的绑定规则。

1. 利用 apply 传入参数数组
```
function foo(a, b) {
    console.log(a, b);
}

foo.apply(null, [1, 2]) // 1 2
```

2. 使用 bind 柯里化函数
```
function foo(a, b) {
    console.log(a, b);
}

const fn = foo.bind(null, 1);

fn(2) // 1 2
```

#### 3.2 间接引用
```
function foo() {
    console.log(this.a)
}

const a = { foo, a: 1 };
a.foo() // 1

let b = a.foo

b() // 严格模式下是 undefined
```
创建一个函数的间接引用，再调用这个间接引用时，会丢失 this 绑定，使用的是默认绑定。


#### 3.3 箭头函数
箭头函数不适用 this 的四个绑定规则，而是根据外层作用域(函数或者全局)作用域来决定 this

```
function foo() {
    return () => {
        console.log(this.a)
    }
}

let b = {
    a: 1
}

let c = {
    a: 2
}

let bar = foo.apply(b)
bar.apply(c) // 1
```

foo 内部创建的箭头函数会捕获调用 foo() 时的this。箭头函数的绑定不能被修改。箭头函数可以像 bind 一样确保函数的this 被绑定到指定对象。

### 参考
[你不知道的JavaScript](https://book.douban.com/subject/26351021/)