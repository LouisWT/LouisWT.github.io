---
layout: simple-article
title: Webpack 与网站性能优化
date: 2018-12-20 23:01:37
categories:
    - 前端
    - Webpack
tags:
    - Webpack
---

- 减少项目体积
    - 代码最小化
    - 利用 loader option 对特殊字符串进行最小化
    - 移除开发期间代码
    - 使用 ES Module 开启摇树优化
    - 图片优化，小图片内联进 app，大图片进行压缩
- 利用缓存
    - 将缓存时间设置很久，如果文件变动，使用文件名来使缓存不命中从而下载新的文件
    - 将项目代码、环境依赖、运行时代码分离。环境依赖很少会变化，可以更好的利用缓存
- 懒加载，开始时仅下载最重要的部分，懒加载剩余区域来提升最初的加载性能
<!-- more -->
这篇文章主要是读了 [利用 webpack 做 web 性能优化](https://beanlee.github.io/2018/05/02/blog-translate-web-performance-optimization-with-webpack-from-google-webpack4/#use-the-production-mode-webpack-4-only-%E4%BD%BF%E7%94%A8%E7%94%9F%E4%BA%A7%E6%A8%A1%E5%BC%8F%E4%BB%85%E7%94%A8%E4%BA%8E-webpack-4) 之后的笔记。

### 一. 减少项目体积

#### 1. 设置 mode 为 production

Webpack 4 增加了一个 mode 配置项，值为 development 或者 production。这个配置项必须写，不然有警告。mode 默认是 production。

设置为 development 后会影响开发阶段:
- 浏览器 debug 工具
- 开发周期内的快速增量编译
- 运行时有用的错误信息

设置为 production 后会影响发布阶段:
- 更小的输出包尺寸
- 运行时高效的代码
- 忽略掉只在开发时启用（development-only）的代码
- 不会暴露源码或者文件路径
- 简化使用打包后资源过程

[具体影响的选项](https://webpack.js.org/concepts/mode/)

[选项详解](https://beanlee.github.io/2018/04/18/blog-translate-webpack-4-mode-and-optimization/)

#### 2. 开启最小化

最小化分为 bundle 最小化 和 利用 loader 特定的 option 最小化。

##### 2.1 bundle 最小化
当 mode 设为 production 后，默认就会开启这个优化，也就是说 `optimization.minimize: true`

开启这个优化后，会对 Webpack 编译后的代理进行压缩：使用更短的变量名；更简洁的写法等等

##### 2.2 利用 loader 特定的 option 最小化
对css代码，webpack会编译成JS文件中的一个字符串，这个字符串需要用 css-loader 才能进行最小化

```
{
    test: /\.css$/,
    use: [
        'style-loader',
         { loader: 'css-loader', options: { minimize: true } },
    ],
}
```

#### 3. 明确 NODE_ENV 移除开发期间代码

设置 mode 为 production 后，会自动设置 `optimization.nodeEnv: production`

这一步会移除开发期间代码；移除掉所有 if (NODE_ENV !== 'production')的条件分支

#### 4. 使用 ES 模块开启摇树优化

使用 ES Modules, 这样 webpack 可以做 tree-shaking，也就是摇树优化，去掉没有用到的依赖

1. 分析 ES Modules 文件的 export 中哪些用到了，哪些没用到。没用到的 在编译后就不会生成 export。(但是没用到的函数或者对象的相关代码依然在)
2. 使用最小化工具移除无用变量，也就是上面那些没有被 export 的变量。

在 .babelrc 中加入 `module: false` 选项，防止 babel 将 ES module 编译为 CommonJS 从而无法优化。

[tree-shaking文档](https://webpack.js.org/guides/tree-shaking/)

#### 5. 图片优化

小图片使用 url-loader svg-loader 打包到 app 中，大图片 使用 image-webpack-loader 来进行优化

### 二. 利用长时间缓存

#### 1. 使用 bundle 版本和缓存头信息

nginx 配置
```
expires 365d;
或
Cache-Control: max-age=31536000
```

webpack 在命名中加入 chunkhash，当文件内容改变后，文件名也会变，缓存不命中所以会下载新的文件
```
  entry: './index.js',
  output: {
    filename: 'bundle.[chunkhash].js',
        // → bundle.8e0d62a03.js
  },
```

#### 2. 将用户源码(bundle)、依赖环境(vender)和 运行时代码(runtime) 分离

一般来说 vender 代码很少会变动，所以将这部分单独抽出来。这样可以充分利用浏览器缓存。

https://webpack.js.org/concepts/manifest/

> runtime 以及 manifest 基本上是在模块化应用程序在浏览器中运行时连接模块化应用程序所需的所有代码。
所谓的runtime，就是帮助 webpack 编译构建后的打包文件在浏览器运行的一些辅助代码段，换句话说，打包后的文件，除了你自己的源码和npm库外，还有 webpack 提供的一点辅助代码段。而 manifest，则是 webpack 用以查找 chunk 真实路径所使用的一份关系表，简单来说，就是 chunk 名对应 chunk 路径的关系表

1. 依赖环境作为 vender 分离出来

`optimization.splitChunks.chunks: 'all'`

2. 将运行时代码 runtime 分离出来

`optimization.runtimeChunk: true`


### 三. 懒加载
通过仅下载最重要的部分，懒加载剩余区域能够提升最初的加载性能。

使用 import() 函数 和 code-splitting 解决这个问题。

1. import()明确表示你期望动态地加载独立的 module。当 webpack 看到 import('./module.js')时，他就会将这个 module 移到独立的 chunk 中。只有到运行到 import() 时才会加载。

2. 代码切分。对于单页面应用，使用路由拆分带页面引用，当加载到需要的路由时，调用 import()，加载所需代码。 对于 React 框架，使用 react-router


### 参考
- [2018 前端性能优化清单](https://juejin.im/post/5a966bd16fb9a0635172a50a)
- [利用 webpack 做 web 性能优化](https://beanlee.github.io/2018/05/02/blog-translate-web-performance-optimization-with-webpack-from-google-webpack4/#use-the-production-mode-webpack-4-only-%E4%BD%BF%E7%94%A8%E7%94%9F%E4%BA%A7%E6%A8%A1%E5%BC%8F%E4%BB%85%E7%94%A8%E4%BA%8E-webpack-4)