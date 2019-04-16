---
layout: simple-article
title: JS小技巧之 Async 异步控制\嵌套属性访问
categories:
    - JavaScript
tags:
    - JavaScript
---

经历了回调地狱和 Promise 的链式调用，现在异步请求主要是靠 async/await 的写法，这篇文章总结了一些 async 在不同使用场景下的写法。
<!-- more -->
### 1. 按顺序执行一系列异步请求
首先上一段代码：
```js
let myData = [{id: 0}, {id: 1}, {id: 2}, {id: 3}]
let res = [];

async function fetchData(dataSet) {
  dataSet.forEach(async (entry) => {
    const result = await axios.get(`https://ironhack-pokeapi.herokuapp.com/pokemon/${entry.id}`)
    const newData = result.data
    res.push(newData)
    console.log(res[res.length - 1].id)
  })
}

fetchData(myData)

// 输出
// 0
// 2
// 1
// 3
// 这个输出结果是不确定的,主要看哪个请求比较快
```
上面这段代码使用了 forEach 来遍历 dataSet,然后发出了一系列的异步请求,但是结果却不能保证异步请求的顺序,这其实是因为 forEach 本身实现只会对每个元素调用传入的函数,这样后面的请求就不会等待之前的请求完成再发送请求,导致不能按照顺序完成这一系列请求
```js
let myData = [{id: 0}, {id: 1}, {id: 2}, {id: 3}]
let res = [];

async function fetchData(dataSet) {
    for(entry of dataSet) {
        const result = await axios.get(`https://ironhack-pokeapi.herokuapp.com/pokemon/${entry.id}`)
        const newData = result.data
        res.push(newData)
        console.log(res[res.length - 1].id)
    }
}
fetchData(myData)
// 输出结果
// 0
// 1
// 2
// 3
```
上面的代码使用了 for...of 来实现对 dataSet 的遍历并发出一系列请求,但是这次是没有出现 forEach 的问题,这是因为 for...of 是通过迭代器来遍历数组的.


for...of 遍历相当于下面的遍历方式:
```js
let myData = [{id: 0}, {id: 1}, {id: 2}, {id: 3}]
let res = [];

async function fetchData(dataSet) {
  let iterator = dataSet[Symbol.iterator]()
  let entry = iterator.next();
  while (!entry.done) {
    const result = await axios.get(`https://ironhack-pokeapi.herokuapp.com/pokemon/${entry.value.id}`)
    const newData = result.data
    res.push(newData)
    console.log(res[res.length - 1].id)
    entry = iterator.next()
  }
}
fetchData(myData)
```

这里可以看到,for...of 这种方式会等当前元素的操作执行完毕后再继续遍历下一个元素,但是这样也带来一个问题,**如果 for...of 中的操作过多,那么可能会导致性能问题**

### 2. 并行执行一系列请求
使用 Promise.all 来并行地执行一系列请求
```js
let myData = [{id: 0}, {id: 1}, {id: 2}, {id: 3}]

async function fetchData(dataSet) {
    const pokemonPromises = dataSet.map(entry => {
        return axios.get(`https://ironhack-pokeapi.herokuapp.com/pokemon/${entry.id}`)
    })
    let res = await Promise.all(pokemonPromises)
    res = res.map(v => v.data)
    console.log(res.map(v => v.id))
}

fetchData(myData)
// 输出
// [0, 1, 2, 3]
```

### 3. 嵌套属性访问
在获得到数据之后,如果需要的数据是一个嵌套的属性,我们很可能要写出这种代码:
```js
let data
if(myObj && myObj.firstProp && myObj.firstProp.secondProp && myObj.firstProp.secondProp.actualData) {
  data = myObj.firstProp.secondProp.actualData
}
```
现在我们可以这样写:
```js
const data = myObj?.firstProp?.secondProp?.actualData
```
简洁好用.但是这是一个 stage-1 的特性,需要 babel 编译,添加 stage-1 的 preset 或者 添加  @babel/plugin-proposal-optional-chaining 来使用这个特性

### 参考链接
[9 Tricks for Kickass JavaScript Developers in 2019](https://levelup.gitconnected.com/9-tricks-for-kickass-javascript-developers-in-2019-eb01dd3def2a)