---
title: webstorm识别webpack别名
date: 2020-12-16 21:01:28
tags:
  - 编译器配置
categories:
  - 编译器配置
---
# webstorm识别webpack别名

> 重装webstorm发现之前配置过的别名识别无了，跳转也跳转不了，在此记录一下

你是否烦恼于设置别名后，webstorm警告并且无法跳转的问题

![image-20201216210710111](image-20201216210710111.png)

<!--more-->

## 步骤

1. 首先创建一个文件webstorm.config.js

```js
'use strict'
const path = require('path')

module.exports = {
  context: path.resolve(__dirname, './'),
  resolve: {
    extensions: ['.js', '.vue', '.json'],
    alias: {
      '@': path.resolve('src'),
      '@assets': path.resolve(__dirname, 'src/assets'),
	  "@components": path.resolve("src/components"),
    }
  }
}
```

2. 然后进入webstorm设置

![image-20201216205942930](image-20201216211532346.png)

然后我们就可以愉快的进行跳转了

![image-20201216210918827](image-20201216210918827.png)