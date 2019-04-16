---
layout: simple-article
title: Chrome dev tools 小结
categories:
    - Chrome
tags:
    - Chrome
---
Chrome 开发者工具非常强大，调试 bug 不在话下，更重要的是让你对网站性能优化不会再无从下手，这篇文章主要总结下常见使用方法
<!-- more -->
### 一. 操作DOM
| 操作类型 | 操作方式 |
| --- | --- |
| 删除DOM节点 | 选中 + delete
| 隐藏DOM节点 | 选中 + H 
| 查找节点 | ctrl + f
| 修改节点属性或者内容 | 双击后修改
| 移动节点位置 |　直接拖动
| 滚动到当前选中元素 |　选中后右键选择　scroll into view
| 引用当前选中元素 | 选中后 $0
| 将当前元素保存为变量 | 选中后右键 Store as global variable
| 让当前元素处于某种状态（比如正被鼠标悬浮的状态） |　选中后 Force state　
| DOM 断点（子树改变、attribure改变、删除节点） | 选中后选择 Break on

### 二. 操作CSS
![](/images/chrome-css.png)


| 操作类型 | 操作方式 |
| --- | --- |
| 给元素添加类 | 样式盘上方 .cls 选项可以给元素添加类
| 让元素处于某种状态（比如鼠标悬浮） | 样式盘上方 .hov 选项，选择一个状态
| 给元素添加新的选择器 | 样式盘上方的 + 选项
| 给元素可视化添加样式 | 每个选择器下方都有三个点，点开后可以看到添加 box-shadow text-shadow background-color 等选项，点开后就可以可视化添加了
| 修改盒模型的宽高属性 |　在样式盘右边的盒模型上的数字可以双击后修改

### 三. 网络面板
![](/images/network.png)

#### 1. 拦截请求
ctrl + shift + p 呼出面板后，输入 block。
文档获取不到被 block 的资源。
![](/images/network-block.png)

### 四. 代码覆盖性检测
![](/images/chrome-coverge.png)

1.  ctrl(command) + shift + p 呼出panel
2.  输入 coverge
3.  在新的框里点击刷新按钮，就可以看到 JS文件和CSS文件中有哪些用到了，哪些没用到



### 五. 查看和修改 CSS 动画
![](/images/chrome-animation.png)
![](/images/chrome-animation2.png)


ctrl(command) + shift + p 呼出panel，然后输入 animation，从而得到动画的 panel

### 六. JS调试
![](/images/chrome-js.png)

### 七. console 窗口的工具函数
| 函数名 | 用法 |
| --- | --- |
| $_ | 引用上次得到的值
| $0 - $4 | 当前选中的DOM节点、上一次选中的DOM节点~上四次选中的DOM节点
| $(<\selector>) | 传入选择器选择一个元素
| $$(<\selector>) | 传入选择器选择所有符合条件的元素
| getEventListeners(<\Object>) | 传入一个DOM节点，返回它上面绑定的事件监听函数
| monitorEvents(<\Object>, [<\事件名>]) | 当对象上触发了对应的事件，将事件打印出来。
| unmonitorEvents(<\Object>, [<\事件名>]) | 取消对象上的事件监听

console对象的三个方法：

| 函数名 | 用法 |
| --- | --- |
| console.table(<\Object>, [<\属性名>]) | 将对象的属性用表格的方式呈现
| console.time(标签名)/timeEnd(相同标签名) | 给操作计时，第二次调用相同标签名后会输出这两次调用时的时间差
| console.count(标签名) | 每次调用这个，都会递增一下

### 八. 性能优化

#### 1. 开启帧率检测

ctrl + shift + p 然后输入 render，打开 FPS meter
![](/images/fps-meter.png)

#### 2. Performance 面板
**使用无痕窗口进行性能评价，从而避免浏览器插件的影响**
![](/images/performance.png)
![](/images/performance.png)

#### 3. 使用audits 来评价网站性能
![](/images/audits.png)
![](/images/audits2.png)


#### 4. 使用 JS profiler 查看火焰图
火焰图也可以在 Performance 的 main 面板中查看。
![](/images/JS-profiler.png)

- 火焰图的横轴表示执行时间，越宽表示执行时间越长，可能需要优化
- 火焰图的纵轴表示调用堆栈，每一层就是一个函数，在当前层的函数中调用另一个函数就会长出下一层

### 九. 进程管理

浏览器 more tools 里面打开任务管理器
![](/images/chrome-task.png)


