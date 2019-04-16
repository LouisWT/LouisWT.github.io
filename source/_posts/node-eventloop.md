---
layout: simple-article
title: 事件循环理解
date: 2018-09-30 21:02:45
categories:
    - 服务器端
    - Node.js
tags:
    - Node.js
---
一直以来对Node.js 的事件循环机制都像蒙了层纱，看过一些文章，也经常只是跑通几个例子，然后强行解释一波，始终无法真正的理解。今天闲逛时看到了[这篇文章](https://cnodejs.org/topic/5a9108d78d6e16e56bb80882#5a98d9a2ce1c90bc44c445af)，感觉讲的很透彻，但是只是看完估计很快就忘了，所以这篇博客就记录下我对这篇文章的理解。
<!-- more -->
### 一. Node.js 的 event loop

#### 1.1 Node.js 与 浏览器的 event loop 区别

- Node.js 的event loop 是基于 libuv 库的；而浏览器的 event loop 是遵循 html5 规范的
- libuv 是事件循环模型的跨平台实现；浏览器的 event loop 每个浏览器厂商的实现都不一样

#### 1.2 参考文档

- [The Node.js Event Loop, Timers, and `process.nextTick()`](https://nodejs.org/en/docs/guides/event-loop-timers-and-nexttick/)
- [libuv文档](http://docs.libuv.org/en/v1.x/design.html)


### 二. Node.js event loop 的运行机制

![data:image](https://qingmooc-v1.oss-cn-qingdao.aliyuncs.com/other/v2-3a59c624e6ff95a7e8c5a23c979f5abe_r.jpg)

上面这个图展示了事件循环的机制，绿色的块是macrotask（宏任务），macrotask 中穿插的粉红箭头是 microtask（微任务）。

#### 2.1 宏任务

宏任务主要有：整体代码(script)、setTimeout、setInterval、I/O、setImmediate

把那个图正过来是下面的样子：
```
   ┌───────────────────────┐
┌─>│        timers         │
│  └──────────┬────────────┘
│  ┌──────────┴────────────┐
│  │     I/O callbacks     │
│  └──────────┬────────────┘
│  ┌──────────┴────────────┐
│  │     idle, prepare     │
│  └──────────┬────────────┘      ┌───────────────┐
│  ┌──────────┴────────────┐      │   incoming:   │
│  │         poll          │<─────┤  connections, │
│  └──────────┬────────────┘      │   data, etc.  │
│  ┌──────────┴────────────┐      └───────────────┘
│  │        check          │
│  └──────────┬────────────┘
│  ┌──────────┴────────────┐
└──┤    close callbacks    │
   └───────────────────────┘
```

- timers: 执行 setTimeout() 和 setInterbal() 中到期的回调函数
- I/O callbacks：上一轮循环中少数的 I/O callback 会被延迟到这一轮的这一阶段执行
- idle,perpre：仅内部使用
- poll: 最重要的阶段，执行 I/O callback，适当的条件下会阻塞在这个阶段
- check: 执行 setImmediate 的回调函数
- close callbacks: 执行close事件的callback，例如socket.on("close",func) 中的 func

**event loop的每一次循环都需要依次经过上述的阶段。  每个阶段都有自己的callback队列，每当进入某个阶段，都会从所属的队列中取出callback来执行，当队列为空或者被执行callback的数量达到系统的最大数量时，进入下一阶段。这六个阶段都执行完毕称为一轮循环。**

#### 2.2 微任务

微任务主要有：Promise.then、process.nextTick

**事件循环的每个阶段都会先执行微任务**。

##### Promise.then(func)
这里指的是 then 中的func 是一个微任务，但是Promise 的构造函数是同步执行的。


### 三. 题目加深理解

这里的题目主要是来源于[知乎专栏](https://zhuanlan.zhihu.com/p/34182184)和[这里](https://github.com/dwqs/blog/issues/61)

这些题目对我理解上面的模型有很大帮助。

#### 3.1 题目一
```
setTimeout(() => {
  console.log('timeout');
}, 0)

setImmediate(() => {
  console.log('immediate');
})

// immediate
// timeout
或
// timeout
// immediate
```
第一个循环：
- 执行 setTimeout，将回调函数注册到 timers 阶段的队列
- 然后执行setImmediate，将回调函数执行到check阶段。
- 然后在 timers 阶段执行 setTimeout的回调，在check阶段执行 setImmediate的回调。

这样看，答案应该是确定的。

但是虽然 timers 阶段在 check 阶段前面，可还有一个问题。

**setTimeout/setInterval 的第二个参数取值范围为[1, 2^31-1]，如果不在这个范围则置为1，所以 setTimeout(func, 0)，实际上是 setTimeout(func, 1)**

- 如果到达 timer阶段的时间距离注册 setTimeout 的回调的时间小于 1ms，那么就不会执行这个回调，所以会先打印 immediate
- 如果到达 timer阶段的时间距离注册 setTimeout 的回调的时间大于 1ms，那么就会执行到期的回调，先打印 timeout

```
setTimeout(() => {
  console.log('timeout');
}, 0)

setImmediate(() => {
  console.log('immediate');
})

let time = Date.now()
while(Date.now() - time < 10);

// timeout
// immediate
```
通过while循环使下个事件循环到达 timer阶段的时间距离注册 setTimeout 的回调的时间一定大于 10ms，这样就肯定会先执行 setTimeout

#### 3.2 题目二
```
const fs = require('fs')

fs.readFile(__filename, () => {
  setTimeout(() => {
    console.log('setTimeout')
  }, 0)

  setImmediate(() => {
    console.log('setImmediate')
  })
})

// setImmediate
// setTimeout
```
第一个循环：
- 首先执行将 IO 的 callback 注册到 poll阶段

不知道第几个循环：
- poll阶段时，发现IO操作完成，执行回调函数。
- 将 setTimeout 的回调注册到 timers 阶段 
- 将 setImmediate 的回调注册到 check 阶段
- IO的回调执行完，从 poll 阶段出来之后，正好是 check 阶段。执行 setImmediate 的回调

之后的一个循环：
- 进入下个事件循环，执行setTimeout 的回调

#### 3.3 题目三
```
setInterval(() => {
  console.log('setInterval')
}, 100)

process.nextTick(function tick () {
  process.nextTick(tick)
})

// 什么都不打印
```
**process.nextTick 内执行 process.nextTick 仍然将 tick 函数注册到当前 microtask 的尾部，所以导致 microtask 永远执行不完**。导致 event loop 上其他 macrotask 阶段的回调函数没有机会执行。

```
setInterval(() => {
  console.log('setInterval')
}, 100)

setImmediate(function immediate () {
  setImmediate(immediate)
})

// 每 100ms 打印一次 setInterval
```
这是因为 **setImmediate 中的setImmediate 会将 immediate 函数注册到下一次 event loop 的 check 阶段，而不是当前正在执行的 check 阶段**，所以给了 event loop 上其他 macrotask 执行的机会。

```
setImmediate(() => {
  console.log('setImmediate1')
  setImmediate(() => {
    console.log('setImmediate2')
  })
  process.nextTick(() => {
    console.log('nextTick')
  })
})

setImmediate(() => {
  console.log('setImmediate3')
})

// setImmediate1
// setImmediate3
// nextTick
// setImmediate2
```
第一个循环：
- 首先会将两个 setImmediate 的回调都注册到下次事件循环的 check 阶段

第二个循环：
- 到check 阶段时，执行第一个 setImmediate 回调，输出 setImmediate1，将 setImmediate2 的回调注册到下个事件循环时，将 process.nextTick 注册到这个宏任务之后的微任务中
- 执行第二个 setImmediate 回调，输出 setImmediate3
- check 阶段宏任务完成，执行微任务，输出nextTick

第三个循环：
- 进入下一个事件循环，输出 setImmediate2

#### 3.4 题目四
```
const promise = Promise.resolve()
  .then(() => {
    return promise
  })
promise.catch(console.error)

// TypeError: Chaining cycle detected for promise #<Promise>

process.nextTick(function tick () {
  process.nextTick(tick)
})
```
promise.then 类似于 process.nextTick，都会将回调函数注册到 microtask 阶段。上面代码会导致死循环。

上面两段代码效果一样。

```
Promise.resolve()
.then(() => {
  console.log('promise');
})

process.nextTick(() => {
  console.log('process');
})

// process
// promise
```
promise.then 虽然和 process.nextTick 一样，都将回调函数注册到 microtask，但优先级不一样。**process.nextTick 的 microtask 队列 总是优先于 promise 的 microtask 队列 执行。**

#### 3.5 题目五
```
setTimeout(() => {
  console.log(1)
}, 0)

new Promise((resolve, reject) => {
  console.log(2)
  for (let i = 0; i < 10000; i++) {
    i === 9999 && resolve()
  }
  console.log(3)
}).then(() => {
  console.log(4)
})

setImmediate(() => {
  console.log(6)
})

console.log(5)

// 2
// 3
// 5
// 4
// 1
// 6
```
第一个循环：
- 将 1的setTimeout 注册到 timers 阶段
- 执行 Promise，打印2、叠加 i，将 then 注册到 microTask、打印3
- 将 6的setImmediate 注册到 check 阶段
- 打印 5
- 执行微任务，打印4
- timers 阶段，打印1
- check阶段，打印5

#### 3.6 题目六
```
setImmediate(() => {
  console.log(1)
  setTimeout(() => {
    console.log(2)
  }, 100)
  setImmediate(() => {
    console.log(3)
  })
  process.nextTick(() => {
    console.log(4)
  })
})
process.nextTick(() => {
  console.log(5)
  setTimeout(() => {
    console.log(6)
  }, 100)
  setImmediate(() => {
    console.log(7)
  })
  process.nextTick(() => {
    console.log(8)
  })
})
console.log(9)

// 9
// 5
// 8
// 1
// 7
// 4
// 3
// 6
// 2
```
第一个循环：
- 首先将 1的setImmediate 的回调注册到下个循环的 check 阶段，将 5的process.nextTick 回调注册到 microTask，然后打印 9
- 执行 microTask，打印 5，将 6的setTimeout 的回调注册到 timers 阶段，将 7的setImmediate 的回调注册到 check 阶段，将 process.nextTick 注册到当前 microTask 队列的尾部
- 当前 microTask 还有任务，执行，打印 8。microTask 队列清空

第二个循环：
- 到达下个循环的timer 阶段，6的setTimeout 的时间还没到，不执行
- 到达 check 阶段，此时队列里有 1的setImmediate 和 7的setImmediate
- 执行 1的setImmediate，打印1，将 2的setTimeout 注册到timers阶段，将 3的setImmediate 注册到下一循环的 check阶段，将 4的process.nextTick 注册到 microTask
- 执行 7的setImmediate，打印7，check 阶段完成
- 执行 microTask 中 4的process.nextTick，打印4

第三个循环：
- 进入下一循环的 timers 阶段，6的setTimeout 与 2的setTimeout 时间都没到，所以不执行
- 到达 check 阶段，执行 3的setImmediate，打印3

不知道第几个循环：
- 等到 setTimeout 到期，执行 6的setTimeout 与 2的setTimeout，打印 6，2

#### 3.7 题目七
```
new Promise(resolve => {
    resolve(1);
    Promise.resolve().then(() => console.log(2));
    console.log(4)
}).then(t => console.log(t));
console.log(3);

// 4
// 3
// 2
// 1
```

第一个循环：
- 同步执行 Promise构造函数，`Promise.resolve` 将 then 的函数注册在 microTask 队列中，打印4，
- 将 1的Promise.then 注册到 microTask 中
- 打印3
- 执行 microTask的第一个任务，打印2
- 执行 第二个任务，打印1


```
Promise.resolve().then(() => {
  console.log(5);
})

new Promise(resolve => {
  resolve(1);
  Promise.resolve().then(() => {
    console.log(2)
  });
  console.log(4)
}).then(t => console.log(t));

process.nextTick(() => {
  console.log(7);
})

Promise.resolve().then(() => {
  console.log(6);
})

console.log(3);

// 4
// 3
// 7
// 5
// 2
// 1
// 6
```
第一个循环:
- 将 5的then 注册到 microTask
- 执行Promise，将 2的then 注册到 microTask，打印4
- 将 Promise的then 注册到 microTask
- 将 7的process.nextTick 注册 microTask
- 将 6的then 注册到 microTask
- 打印3
- 执行 microTask，先执行 7的process.nextTick，打印7
- 执行 Promise 的 microTask，打印 5，2，1，6

```
new Promise(resolve => {
  resolve(1);
  Promise.resolve().then(() => {
    console.log(2)
    process.nextTick(() => {
      console.log(5);
    });
  });
  Promise.resolve().then(() => {
    console.log(6);
  })
  console.log(4)
}).then(t => console.log(t));
console.log(3);

// 4
// 3
// 2
// 6
// 1
// 5
```
第一个循环：
- 将 2的then 注册到 microTask
- 将 6的then 注册到 microTask
- 打印4
- 将 Promise的then 注册到 microTask
- 打印3
- 执行 microTask中Promise的队列， 先执行 2的then，打印2，将process.nextTick注册到microTask
- 执行 6的then，打印6
- 执行 Promise的then，打印1。
- 执行 process.nextTick ，打印5

### 四.要点总结
1. 事件循环模型。

2. 事件循环的每个阶段都会先执行微任务

3. 细节：
- Promise 的构造函数是同步执行的，then 回调才会进 微任务队列。Promise 执行到 then语句，才会把 then 中的回调排进 微任务队列
- process.nextTick 会将函数注册到微任务队列
- process.nectTick 和 Promise 的微任务是两个队列。且 process.nectTick 的优先级高
- setImmediate 中的setImmediate 会将 immediate 函数注册到下一次 event loop 的 check 阶段，而不是当前正在执行的 check 阶段
- setTimeout/setInterval 的第二个参数取值范围为[1, 2^31-1]，如果不在这个范围则置为1，setTimeout(func, 0)，实际上是 setTimeout(func, 1)。

### 五. 参考

- [不要混淆nodejs和浏览器中的event loop](https://cnodejs.org/topic/5a9108d78d6e16e56bb80882#5a98d9a2ce1c90bc44c445af)
- [Event Loop 必知必会（六道题）](https://zhuanlan.zhihu.com/p/34182184)
- [从一道题浅说 JavaScript 的事件循环](https://github.com/dwqs/blog/issues/61)