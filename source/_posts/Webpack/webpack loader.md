---
title: webpack loader
date: 2020-11-27 22:19:51
tags:
  - Webpack
categories:
  - 前端
---
关于loader的作用和配置方法，在 初识webpack 这篇文章中已经讲过，本篇文章会讲常用的一些loader，并自己实现一个loader函数。
<!--more-->
## 1. 转义es6

### 安装

```npm
npm i @babel/core babel-loader @babel/preset-env -D
```

- babel-loader：它是使babel与webpack协同工作的模块
- @babel/core：是babel编译器的核心模块
- @babel/preset-env：官方推荐的预置器，可根据用户设置的目标环境自动添加所需的插件和补丁来编译es6代码

### 配置

```js
rules: [
    {
        test: /\.js$/,
        use: {
            loader: 'babel-loader',
            // 配置选项里的presets
            // 包含ES6还有之后的版本和那些仅仅是草案的内容
            options: {
                cacheDirectory: true, // 启动缓存机制，在重复打包未改变过的模块时防止二次编译，加快打包速度
                presets: ['@babel/preset-env']
            }
        }
        include: /src/,          // 只转化src目录下的js
        exclude: /node_modules/  // 排除掉node_modules，优化打包速度
    }
]
```

## 2. 转义Ts

### 安装

```npm
npm i ts-loader typescript
```

### 配置

```js
rules: [
    {
        test: /\.ts$/,
        use: 'ts-loader',
]
```

与寻常loader不同的是，ts配置项是在工程目录下的`tsconfig.json`中

## 3. html-loader

用于将HTML文件转化为字符串并进行格式化，这使得外面可以把一个HTML片段通过js加载进来。

### 安装

```npm
npm i html-loader
```

### 配置

```js
rules: [
    {
        test: /\.html$/,
        use: 'html-loader',
]
```

## 4. file-loader

用于打包文件类型的资源，并返回其publicPath

### 安装

```npm
npm i file-loader
```

### 配置

```js
rules: [
    {
        test: /\.(jpe?g|png|gif)$/,
        use: 'file-loader',
]
```

## 5. url-loader

### 安装

```npm
npm i url-loader
```

### 配置

```js
rules: [
    {
        test: /\.(jpe?g|png|gif)$/,
        use: {
            loader: 'url-loader',
            options: {
                limit: 10240, // 如果小于该配置大小，则返回文件的base64形式
                name: '[name].[ext]',
                publicPath: './assets-path/', // 会覆盖webpack配置的publicPath
            },
        },
]
```

## 6. 加载样式

### 安装

```npm
npm i style-loader css-loader less-loader -D
```

### 配置

```js
rules: [
    {
        test: /.less$/,
        use: [
            'style-loader', // 将样式通过<style>标签插入到head中
            'css-loader', // 用于加载.css文件，并转化成commonjs对象
            'less-loader' // 将less转化成css
        ]
    },
]
```

## 7. 自定义loader

我们将实现一个loader，它会为所有JS文件开启严格模式，也就是说它会在文件头部加上如下代码

```js
'user strict'
```

force-strict-loader.js

```js
// 提供一些帮助函数，这里用于获取options中的配置项
var loaderUtils = require("loader-utils");
var SourceNode = require('source-map').SourceNode;
var SourceMapConsumer = require('source-map').SourceMapConsumer;

module.exports = function (content, sourceMap) {
    var useStrictPrefix = '\'use strict\';\n\n';
    // 开启缓存，如果文件输入和其依赖没有发生变化时，应该让loader直接使用缓存，提高webpack打包的速度
    if (this.cacheable) {
        this.cacheable();
    }
    // 开启source-map可以便于实际开发者在浏览器控制台查看源代码
    // 如果没有处理，最终无法生成正确的map文件，在dev tool中可能看到错乱的源码
    var options = loaderUtils.getOptions(this) || {};
    // 只有在从配置中获取sourceMap或者从上一个loader中传递下来才会继续处理
    if (options.sourceMap && sourceMap) {
        var currentReguest = loaderUtils.getCurrentRequest(this);
        var node = SourceNode.formStringWithSourceMap(
            content,
            new SourceMapConsumer(sourceMap)
        );
        node.prepend(useStrictPrefix);
        var result = node.toStringWithSourceMap({ file: currentReguest});
        var callback = this.async();
        callback(null, result.code, result.map.toJSON());
    }
    return useStrictPrefix + content;
}
```

上面的插件提供了source-map的配置选项，source-map是一个信息文件，里面储存着位置信息，在debug的时候就会显示原文件的信息和位置，而不是转化后的信息。

使用方法

```js
use: {
    loader: 'force-strict-loader',
    options: {
        sourceMap: true,
    }
}
```

## 参考

《Webpack实战入门、进阶与调优》
https://juejin.cn/post/6844903599080734728#heading-14
http://www.ruanyifeng.com/blog/2013/01/javascript_source_map.html