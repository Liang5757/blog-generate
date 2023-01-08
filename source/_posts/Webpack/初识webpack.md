---
title: 初识webpack
date: 2020-11-22 22:34:14
tags:
  - Webpack
categories:
  - 前端
---

## 1. 安装

```bash
npm install webpack webpack-cli --save-dev
```

webpack是核心模块，webpack-cli则是命令行工具

然后在根目录下创建webpack.config.js来配置webpack
<!--more-->
## 2. 配置资源入口

在配置入口的时候，实际上是做了两件事：

- 确定入口模块的位置，告诉webpack从那里开始打包
- 定义`chunk name`，如果只有一个入口，默认`chunk name`为“main”，如果为多入口，则需要为每一个入口定义`chunk name`作为其唯一标识

### 2.1 context

> entry路径前缀

```js
module.exports = {
    context: path.join(__dirname, './src'), // 资源入口前缀（为了让entry编写的更简洁
    entry: './index.js', // string | object | array | function
};
```

### 2.2 entry

支持四种类型的值

1. 字符串类型

	直接传入文件路径（绝对路径 | 相对路径）

2. 数组类型

	作用是将**多个资源预先合并**

	```js
	module.exports = {
	    entry: ["babel-polyfill", "./src/index.js"], // 最后的文件作为入口
	};
	```

3. 对象类型

	想要定义多入口，则必须使用对象的形式

	 ```js
	module.exports = {
	    entry: {
	        // chunk name为index，入口路径为./src/index.js
	        index: ["babel-polyfill", "./src/index.js"],
	        // chunk name为lib1，入口路径为./src/lib.js
	        lib1: './src/lib.js',
	    }
	};
	 ```

4. 函数类型

	可以动态设置文件路口，甚至加上异步逻辑

	```js
	module.exports = {
	    entry: () => new Promise((resolve) => {
	        setTimeout(() => {
	            resolve('./src/index.js');
	        }, 1000)
	    })
	};
	```

### 2.3 vender

一般是把工程所使用的库、框架等第三方模块集中打包而产生的bundle，可以使用CommonsChunkPlugin将app与vendor这两个chunk中的公共模块提取处理，使app.js产生的bundle将只包含业务模块。

```js
module.exports = {
    entry: {
        app: './src/app.js',
        vendor: ['vue'],
    }
};
```

## 3. 配置资源出口

### 3.1 filename

作用是控制**输出资源的文件名**，也可以为相对路径 和 模板变量（用于控制客户端缓存，下一篇博客将会讲文件指纹）

```js
module.exports = {
    entry: './src/app.js',
    output: {
        filename: "bundle.js"
    }
};
```

### 3.2 path

可以指定资源输出的位置，要求值必须为绝对路径

```js
module.exports = {
    entry: './src/app.js',
    output: {
        filename: "bundle.js",
        path: path.join(__dirname, 'dist'),
    }
};
```

### 3.3 publicPath

`path`用来指定资源输出位置，而`publicPath`用来指定资源请求位置

```js
// 假设当前地址为 https://example.com/app/index.html
// 异步加载的资源名为 0.chunk.js
publicPath: "./js" // 实际路径 https://example.com/app/js/0.chunk.js
```

## 4. loader

webpack 开箱即用只支持 JS 和 JSON 两种文件类型，通过 Loader 去支持其它文件类型并且把它们转化成有效的模块，并且可以添加到依赖图中。

```js
module.exports = {
    // ...
    module: {
        rules: [{
            enforce: normal // pre: 在所有loader之前调用 | normal | post: 在所有loader之后调用
            test: /\.css$/, // 可以接受正则表达式或者正则数组
            use: ['css-loader', 'less-loader'], // 可以接收一个数组，包含匹配项所使用loader，从右向左处理
            exclude: /src\/lib/, // 排除目录
            include: /src/, // 包含目录，exclude和include同时存在时-exclude优先级更高
            issuer: { // 上面是对被加载者的配置，这是加载者的配置
				test: /\.js$/, // 该配置的意思为只有 /src/pages目录下的js可以引用css
            	 include: /src/pages/,
            }
        }]
    }
};
```

## 5.plugins

对webpack loader的扩展，作用于构建的整个过程

```javascript
plugins: [
    new HtmlWebpackPlugin({template: './src/index.html'})
]
```

## 6. mode

| 选项          | 描述                                                         |
| ------------- | ------------------------------------------------------------ |
| `development`                               | 设置`process.env.NODE_ENV`的值为`development`开启`NamedChunksPlugin`和`NamedModulesPlugin` |
| `production`                                  | 设置`process.env.NODE_ENV`的值为`production`开启`FlagDependencyUsagePlugin`，`FlagInc ludedChunksPlugin`，                       `ModuleConcatenationPlugin`，`NoEmitOnErrorsPlugin`，`OccurrenceOrderPlugin`， `SideEffectsFlagPlugin`和`TerserPlugin` |
| `none`                                          | 不开启任何优化选项                                           |

