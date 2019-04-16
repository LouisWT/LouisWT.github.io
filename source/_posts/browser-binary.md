---
layout: simple-article
title: 浏览器的 Blob 对象
categories:
    - Chrome
tags:
    - Chrome
---
浏览器中使用 Blob 对象来表示一个类文件对象，File 对象继承自 Blob，我感觉就类似一个文件描述符。
<!-- more -->

# 一. 转换关系
![](/images/browser-binary.jpg)

# 二. 相关Web API

## 1. Blob 对象
Blob 对象表示一个不可变的、原始数据的类文件对象。

File 接口基于 Blob，继承了 Blob 的功能并将其拓展使其支持用户文件系统的文件。

### 方法
#### 1. 构造函数 `Blob()`
`Blob()`构造函数返回一个新的 Blob 对象。blob 的内容由参数数组中给出的值串联组成。
```js
let blob = new Blob(likeArray, options);
```
- likeArray 是一个由 `ArrayBuffer、ArrayBufferView(TypedArray、DataView)、Blob 和 DOMString` 等对象组成的 Array
- options:
  - type: blob 对象的 mime 类型

**可以使用它来拼接多个 blob**
#### 2. `Blob.prototype.slice(start, [end, [contentType]])`
截取当前 Blob 生成新 Blob。

### 属性
- size: blob对象的字节数
- type: blob对象的MIME类型


### 用法：
- 从其他非 blob 对象构造一个 blob 或使用多个 blob 拼接一个 blob，使用 `Blob()` 构造函数
- 创建包含另一个 Blob 部分数据的 blob，使用 `slice()` 方法
- 获取用户文件系统上的文件对应的 blob 对象(实际上是 File 对象)，一般使用`type='file'`的 input 元素

## 2. File 对象
File 对象继承自 Blob 对象，依然有 slice 方法，只是属性更多

### 属性
- size: File 对象的字节数
- type: File 对象的MIME类型
- name: 文件名称
- lastModified: 当前File 对象最后的修改时间，时间戳形式，单位是 ms

## 3. FileReader 对象
要想读取 Blob 对象的内容，必须使用 FileReader 对象。

FileReader 对象可以异步读取用户操作系统上面的文件（File 对象或者 Blob 对象）内容

### 方法

#### 1. 构造函数
```js
let reader = new FileReader()
```

#### 2. `FileReader.prototype.abort()`
中止读取操作，返回时，readyState 为 DONE

#### 3. `FileReader.prototype.readAsArrayBuffer(blob)`
读取指定 blob 的内容，读取完成后，result 属性是 blob 的 ArrayBuffer 数据对象

#### 4. `FileReader.prototype.readAsDataURL(blob)`
读取 blob 的内容，完成后 result 属性中包含一个 `data:` URL 格式的字符串表示所读取文件的内容

#### 5. `FileReader.prototype.readAsText(blob)`
读取 blob 内容，完成后 result 属性包含字符串形式的文件内容

### 属性
- error: 表示读取文件发生的错误
- result: 读取完成后，文件的内容
- readyState:
  - 0 表示还没有任何数据
  - 1 数据正在加载
  - 2 已经读取完成

### 事件处理

#### 1. `FileReader.onabort`
处理 abort 事件，当读取文件中断时触发

#### 2. `FileReader.onerror`
处理 error 事件，出现错误时触发

#### 3. `FileReader.onloadstart/onprogress/onloadend/onload`
- onloadstart 读取开始
- onprogress 读取中
- onloadend 读取结束
- load 处理 load 事件，在读取操作完成后触发

### 使用 FileReader 读取 blob(Promise 封装)
```js
async function readBlob(blob, type) {
  return new Promise((resolve, reject) => {
    let reader = new FileReader()
    reader.onload = function () {
      resolve(reader.result)      
    }
    reader.onerror = function () {
      reject(reader.error)
    }
    switch(type) {
      case 'ArrayBuffer': reader.readAsArrayBuffer(blob)
        break
      case 'DataURL': reader.readAsDataURL(blob)
        break
      case 'Text': reader.readAsText(blob)
        break
    }
  })
}
```