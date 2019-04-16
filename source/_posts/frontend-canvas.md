---
layout: simple-article
title: Canvas 之 2D 上下文
date: 2018-05-11 17:05:20
categories:
    - 前端
    - Canvas
tags:
    - Canvas
---

Canvas 可以在页面设定一个区域，然后就可以通过JS动态地在这个区域绘制图形。

要在 Canvas 上绘图，需要取得绘图上下文。Canvas 有 2D 上下文 和 WebGL 3D 上下文。这篇文章主要讲2D 上下文

<!-- more -->

## 1. 获取 2D 上下文
使用`getContext('2d')`获取2D上下文
```js
let canvas = document.getElementById('canvas')
let ctx = canvas.getContext('2d')
```
使用 2D 上下文提供的方法，就可以绘制简单的2D图形了。

## 2. 2D 上下文
2D上下文的坐标开始于 canvas 元素的左上角，原点是 (0,0)，x 表示横坐标，越大越靠右，y表示纵坐标，越大越靠下; width 表示水平方向的像素个数，height 表示垂直方向的像素个数。

### 2.1 填充和描边
**两种基本的绘图操作就是填充和描边**：
- 填充：**用指定样式(颜色，渐变，图像)填充图形**
- 描边：**只在图形的边缘画线**

填充的结果取决于 `fillStyle`；
描边的结果取决于 `strokeStyle`

这两个值默认都是 #000000

#### 2.1.1 描边相关属性
- 描边宽度: lineWidth
- 描边末端形状：lineCap 取值 butt | round | square

![data:image](https://mdn.mozillademos.org/files/236/Canvas_linecap.png)

线的末端该如何处理：
从左到右分别是 平头(什么都不加) 圆头(加一个半圆) 方头(加一个矩形)
- 线条相交方式：lineJoin 取值 round | bevel | miter

![data:image](https://mdn.mozillademos.org/files/237/Canvas_linejoin.png)

两个不同方向的线，连接处存在缝隙，如何去补。
从左到右分别是 圆交(补一个扇形) 斜交(补一个三角形) 斜接(补一个菱形)

示例：
```js
ctx.lineWidth = 10;
ctx.lineJoin = "miter";
ctx.beginPath();
ctx.moveTo(0,0);
ctx.lineTo(200, 100);
ctx.lineTo(300,0);
ctx.stroke();
```

### 2.2 绘制矩形
- fillRect(x, y, width, height): 填充一个矩形
- strokeRect(x, y, width, height): 描一个矩形的框
- clearRect(x, y, width, heigth): 清除画布上的一块矩形区域

### 2.3 绘制路径
1. 调用 beginPath(), 表示要开始画新路径了
2. 绘制路径
  - arc 绘制弧线
  - arcTo 从上一点绘制弧线
  - bezierCurveTo 从上一点绘制贝塞尔曲线
  - lineTo 从上一点绘制一条直线
  - moveTo 将绘图游标移动到另一点，不画线
  - quadraticCurveTo 从上一点绘制一条二次曲线
  - rect 绘制一个矩形路径
3. 创建路径后可以进行如下操作
  - closePath: 绘制一个连接起点到终点的整个的路径
  - fill: 对路径填充
  - storke: 对路径描边

绘制路径是很多复杂图形的基础，这里没有详细介绍每个 API，只是对绘制路径的流程有个大致的认识，具体使用的时候可以去 MDN 仔细查阅文档。

### 2.4 生成图像和绘制图像
使用 canvas 的 toDataUrl 方法可以导出canvas 元素上绘制的图像。
```js
let canvas = document.getElementById('canvas')
// 传入的参数为图像的 MIME 类型格式
let imgUri = canvas.toDataURL('image/png')
let image = document.createElement('img');
image.src = imgUri;
```
使用 2D 上下文的 drawImage，可以在画布上绘制图像；可以向其传递 img 元素 或者 另一个 canvas 元素，如果传递的是另一个 canvas 元素，那么可以将另一个画布的内容绘制到当前画布
```js
// 绘制的起点, 图像的大小与原始的一样
ctx.drawImage(image | canvas, x, y)

// 除了绘制起点，还传入了绘制的图像的大小
ctx.drawImage(image | canvas, x, y, width, height)

// 将源图像的某个区域绘制过来
ctx.drawImage(image | canvas, sourceX, sourceY, sourceWidth, sourceHeight, tarX, tarY, tarWidth, tarHeight)
```

目前项目中用到的 Canvas 特性差不多就这些，基本上可以算作入门了，我相信学习是要用实践作为动力的，接下来如果用到了更高级的特性，我会再来完善的 :)