---
layout: simple-article
title: fabric.js 入门
date: 2018-06-17 19:02:37
categories:
    - 前端
    - Canvas
tags:
    - fabric.js
---
html5 的 canvas 元素可以实现许多惊艳的效果，但是使用原生canvas开发十分痛苦，因为canvas的绘制api是非常低级的，在绘制时需要运用各种技巧，并且代码的逻辑性也不够强。

fabric.js 是一个 Canvas 的高级抽象库，它提供了 canvas 没有的对象模型、SVG解析器、交互层以及一些工具。
<!-- more -->

## 1. 为什么要使用 Fabric.js 

编写原生的 canvas 非常痛苦，因为原生 canvas 提供的 api 比较低级，是一些简单的图形命令。比如画一个矩形，使用`fillRect(left, top, width, height)`；画一条线，使用`moveTo(left, top)` 和 `lineTo(left, top)`，这些 api 让我们编写程序就像画画，需要一步一步地画，没有一个更高级的抽象。

Fabric 在原生方法的基础之上提供了简单强大的对象模型。Fabric 使我们可以直接使用对象来进行绘制操作。

### 1.1 画矩形的对比
下面是使用原生 Canvas 和 Fabric 画一个矩形的对比：
```js
// 1. 使用原生 Canvas 画一个矩形
// 获取 canvas 元素
var canvasEl = document.getElementById('canvas');
// 2d 画布
var ctx = canvasEl.getContext('2d');
// 填充颜色
ctx.fillStyle = 'red';
// 在 100，100 处画一个 20*20 的矩形
ctx.fillRect(100, 100, 20, 20);


// 2. 使用 fabric
// 给原生的 canvas 创建一个 wrapper
var canvas = new fabric.Canvas('canvas');
// 创建一个 100,100 处 20*20，填充颜色为红色的矩形
var rect = new fabric.Rect({
  left: 100,
  top: 100,
  fill: 'red',
  width: 20,
  height: 20
});
// 向画布中添加矩形
canvas.add(rect);
```
可以看到，原生canvas的绘制主要使用 2d context 的 api，而 fabric 则是使用对象进行绘制：实例化一个矩形，设置对应的属性，将矩形加入画布。

### 1.2 加45度旋转的对比

目前看上去，fabric 的方式尽管看上去更直观，但是没有省多少代码，下面加一个旋转45度的效果
```js
// 1. 原生方式
var canvasEl = document.getElementById('canvas');
var ctx = canvasEl.getContext('2d');
ctx.fillStyle = 'red';
ctx.translate(100, 100);
ctx.rotate(Math.PI / 180 * 45);
ctx.fillRect(-10, -10, 20, 20);


// 2. fabric 方式
var canvas = new fabric.Canvas('c');
var rect = new fabric.Rect({
  left: 100,
  top: 100,
  fill: 'red',
  width: 20,
  height: 20,
  // 指定一个 45 度旋转
  angle: 45
});
canvas.add(rect);
```
使用原生canvas需要结合使用 translate 和 rotate 才能实现旋转，而 fabic 中则指定 angle 属性就可以了。fabric 相当于抽象了旋转这个操作，避免了样板代码。 

### 1.3 移动矩形的对比
```js
// 1. canvas 实现
var canvasEl = document.getElementById('canvas');
var ctx = canvasEl.getContext('2d');
ctx.fillStyle = 'red';
ctx.fillRect(100, 100, 20, 20);
// 先擦除画布的矩形
ctx.clearRect(0, 0, canvasEl.width, canvasEl.height);
// 画新的矩形
ctx.fillRect(20, 50, 20, 20);

// 2. fabric 实现
var canvas = new fabric.Canvas('canvas');
var rect = new fabric.Rect({
  left: 100,
  top: 100,
  fill: 'red',
  width: 20,
  height: 20
});
canvas.add(rect);
// 直接移动
rect.set({left: 20, top: 50})
canvas.renderAll()
```
原生的 canvas 如果直接调用第二个 fillRect，那么只会在原来画布的基础上再画一个矩形，而不会移动现有的矩形，所以要有一个清空画布的操作；而通过 fabric，只要改变对象的属性，重新渲染就好了。

## 2 对象模型
Fabric 包含了所有常见的基础图形类，主要有以下7类：
- fabric.Circle 圆
- fabric.Ellipse 椭圆
- fabric.Line 直线
- fabric.Polygon 多边形
- fabric.Polyline 折线
- fabric.Rect 矩形
- fabric.Triangle 三角形

绘制时，只要创建一个类的实例，并将其加入 canvas 就好了。

### 2.1 对象属性
上面移动矩形的例子中，使用了 `rect.set({left: 50, top: 20})`，通过 set 方法改变对象属性，重渲染后就得到了矩形移动的效果。

对象都有以下的属性可以set设置：
- 位置：left、top
- 大小：width、height
- 渲染：fill、opacity、stroke、strokeWidth
- 缩放：scaleX、scaleY
- 旋转：angle
- 翻转：flipX、flipY
- 歪斜：skewX、skewY

可以使用 get('name') 或者 getName() 这两种方式来获取对象属性，比如 get('left') 或 getLeft()。

### 2.2 继承关系
大部分的图形类都继承自 fabric.Object。fabric.Object 基本就代表一个二维图形

## 3. fabric.Canvas

fabric.Canvas 是原生canvas 的包装器，它负责管理画布上的所有结构对象。

使用方法如下：
```js
// el 是 canvas 元素
// string 是 ID 选择器
let canvas = new fabric.Canvas(el | string, options)
// 使用 add 方法添加图形对象
canvas.add(rect)
```
### 3.1 Canvas 与 StaticCanvas 的不同与联系

Canvas 类是继承自 StaticCanvas 类，不同在于 Canvas 类多了一个交互层，允许用户用鼠标与 canvas 上的对象交互。如果不需要用户交互，那么可以使用 StaticCanvas 类。