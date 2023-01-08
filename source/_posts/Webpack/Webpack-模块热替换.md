---
title: Webpack 模块热替换
date: 2020-12-11 23:36:39
tags:
  - Webpack
categories:
  - 前端
---
> 让代码在网页不刷新的前提下得到最新的改动，这就是Hot Module Replacement，HMR
<!--more-->
## 1. 开启HMR

配置文件

```js
const webpack = require('webpack')

module.exports = {
    plugins: [
        new webpack.HotModuleReplacementPlugin()
    ],
    devServer: {
        hot: true,
    },
}
```

然后在应用的入口index.js下调用HMR API，这样HMR对于index.js及其依赖的所有模块都会生效

```js
// index.js
if (module.hot) {
	module.hot.accept();
}
```

## 2. 原理

> 在本地开发环境下，浏览器是我们的客户端，webpack-dev-server（WDS）相当于我们的服务端，核心就是客户端从服务端（WDS）拉取更新后的资源（HMR拉取的不是整个资源文件，而是chunk diff，即chunk需要更新的部分）

实际上WDS与浏览器之间建立了一个**websorket**，当本地资源发生变化时WDS会向浏览器推送更新事件，并带上这次构建的hash，让客户端与上一次资源进行比对可以防止冗余更新的出现。

![image-20201211231509387](image-20201211231509387.png)

现在客户端根据hash比对后知道新的构建结构和当前的有了差别，就会向WDS发起一个请求来获取更改文件的列表，通常这个请求的名字为`[hash].hot-update.json`

![image-20201211232331368](image-20201211232331368.png)

![image-20201211232346317](image-20201211232346317.png)

该返回结构告诉客户端，需要更新的chunk为`18`，版本为（构建hash）2ef4d1d91c537e43ce76。这样客户端就可以再借助这些信息继续向WDS获取该chunk的增量更新。

![image-20201211232412630](image-20201211232412630.png)

![image-20201211232443976](image-20201211232443976.png)

现在客户端已经获取到chunk的更新，现在还有一个问题，客户端获取到这些增量更新之后如何处理，这就不属于webpack的工作了，但是HMR提供了相关的api。官方api文档：https://www.webpackjs.com/api/hot-module-replacement/

可以看到HMR是使用了`webpackHotUpdate`来处理的，执行 `webpackHotUpdate` 时如发现模块代码实现了 HMR 接口，就会执行相应的回调或者方法，从而达到应用更新时，模块可以自行管理自己所需要额外做的工作。

这里还有一个问题是，webpack 如何保证 HMR 接口中的引用是最新的模块代码？我们看一个简单的例子：

```js
import './index.css'
import hello from './bar'

hello()

if (module.hot) {
  module.hot.accept('./bar', () => {
    // console.log('Accepting the updated bar module!')
    hello()
  })
}
```

从代码上看，hello 都是同一个，这样的话并没有办法引用最新的模块代码，但是我们看一下上述代码在 webpack 构建后的结果：

```js
if (true) {
  module.hot.accept("./src/bar.js", function(__WEBPACK_OUTDATED_DEPENDENCIES__) { 
    /* harmony import */ 
    __WEBPACK_IMPORTED_MODULE_1__bar__ = __webpack_require__("./src/bar.js"); 
    (() => {
      // console.log('Accepting the updated bar module!')
      Object(__WEBPACK_IMPORTED_MODULE_1__bar__["default"])()
    })(__WEBPACK_OUTDATED_DEPENDENCIES__); 
  })
}
```

其他代码比较杂，我们集中看 `module.hot` 的处理部分。这里可以发现，我们的 hello 已经重新使用 `__webpack_require__` 来引用了，所以可以确保它是最新的模块代码。

## 参考

《Webpack实战》
掘金小册：《使用 webpack 定制前端开发环境》