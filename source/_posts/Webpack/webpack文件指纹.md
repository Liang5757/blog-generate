---
title: webpack文件指纹
date: 2020-11-22 22:35:32
tags:
  - Webpack
categories:
  - 前端
---
> 打包后的文件后缀，通常用于做版本管理，文件被修改后打包出来的文件指纹不同，浏览器只会下载这些不同的文件，没被修改的文件从缓存读取，加快浏览速度
<!--more-->
## 1. 三种文件指纹的策略

| 文件指纹    | 策略                                                       |
| ----------- | ---------------------------------------------------------- |
| hash        | 只要有文件被修改，每一次构建过程生成唯一hash，所有文件共用 |
| chunkhash   | 基于chunk内容生成，不同的entry会生成不同的chunkhash值      |
| contenthash | 根据文件内容来定义hash，文件内容不变，则contenthash不变    |

chunkhash的问题：如果在js中引入了style文件，之后只修改了style文件，但打包后js的文件指纹也一起修改了

## 2. 文件指纹的设置

### 2.1 js的文件指纹设置

 js使用chunkhash型的文件指纹

```js
module.exports = {
    entry: {
        app: './src/app.js',
        search: './src/search.js'
    },
    output: {
        filename: '[name][chunkhash:8].js', // 取chunkhash的前8位
        path: __dirname + '/dist'
    }
};
```

### 2.2 css的文件指纹设置

```js
module.exports = {
    entry: {
        app: './src/app.js',
        search: './src/search.js'
    },
    output: {
        filename: '[name][chunkhash:8].js',
        path: __dirname + '/dist'
    },
    plugins: [
        // 如果使用style-loader设置解析样式，做不到各自独立的css文件
        // 所以使用 MiniCssExtractPlugin 插件抽离出各个独立的css文件
        new MiniCssExtractPlugin({
            filename: `[name][contenthash:8].css`
        });
    ]
};
```

### 2.3 图片的文件指纹

```js
const path = require('path');
module.exports = {
    entry: './src/index.js',
    output: {
        filename: 'bundle.js',
        path: path.resolve(__dirname, 'dist')
    },
    module: {
        rules: [
            {
                test: /\.(png|svg|jpg|gif)$/,
                use: [{
                    loader: 'file-loader’,
                    options: {
                    name: 'img/[name][hash:8].[ext]'
                	}
                }]
            }
        ]
    }
};
```

