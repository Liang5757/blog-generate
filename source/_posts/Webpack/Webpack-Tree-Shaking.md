---
title: Webpack Tree Shaking
date: 2020-12-11 01:44:53
tags:
  - Webpack
categories:
  - 前端
---
> 由Rollup提出，为了消除无用的JavaScript代码而被引入的，对于es模块依赖关系确定的，就可以进行静态分析，CJS不可以
<!--more-->
## 1. 作用对象

- 不会被执行、不可到达的代码
- 代码执行的结果不会被用到
- 代码只会影响死变量（只写不读）

## 2. 作用方式

> tree shaking 只会对满足上述条件的代码进行**注释标记**，由`UglifyJSPlugin`清除

webpack 负责对代码进行标记，把`import`&`export`标记为 3 类：

1. 所有`import`标记为`/* harmony import */`
2. 被使用过的`export`标记为`/* harmony export ([type]) */`，其中`[type]`和 webpack 内部有关，可能是`binding`, `immutable`等等。
3. 没被使用过的`export`标记为`/* unused harmony export [FuncName] */`，其中 `[FuncName]`为`export`的方法名称

## 3. 不会清除类和IIFE

下面摘取了rollup核心贡献者的的一些回答：

![img](160bfd6bef36c293)

- rollup只处理函数和顶层的import/export变量，不能把没用到的类的方法消除掉
- javascrip**t动态语言的特性使得静态分析比较困难**
- 下部分的代码就是副作用的一个例子，如果静态分析的时候删除里run或者jump，程序运行时就可能报错，那就本末倒置了，我们的目的是优化，肯定不能影响执行

下面这篇文章介绍了Babel编译类文件导致副作用，以及UglifyJs贡献者对处理有副作用代码的态度
https://juejin.cn/post/6844903549290151949 (你的Tree-Shaking并没什么卵用)

## 4. 对于IIFE返回的函数，如果未被使用则会清除

```js
//App.js
import { cube } from './utils.js';
console.log(cube(2));

//utils.js
var square = function(x) {
  console.log('square');
  return x * x;
}();

function getSquare() {
  console.log('getSquare');
  square();
}

export function cube(x) {
  console.log('cube');
  return x * x * x;
}
```

结果

```js
function(e, t, n) {
  "use strict";
  n.r(t);
  console.log("square");  // square这个IIFE内部的代码还在
  console.log(function(e) {
    return console.log("cube"), e * e * e  
    // square这个IIFEreturn的方法因为getSquare未被调用而被删除
  }(2))
}
```

## 5.Tree shaking结合第三方包使用

```js
//App.js
import { getLast } from './utils.js';
console.log(getLast('abcdefg'));

//utils.js
import _ from 'lodash';  // 这里的引用方式不同，会造成bundle的不同结果

export function getLast(string) {
  console.log('getLast');
  return _.last(string);
}
```

结果

```js
import _ from 'lodash';
    Asset      Size 
bundle.js  70.5 KiB

import { last } from 'lodash';
    Asset      Size
bundle.js  70.5 KiB

import last from 'lodash/last';   // 这种引用方式明显降低了打包后的大小
    Asset      Size
bundle.js  1.14 KiB
```

## 6. tree shaking结合第三方包使用

```jS
//App.js
import { Add } from './utils'
Add(1 + 2);

//utils.js
import { isArray } from 'lodash-es';

export function array(array) {
  console.log('isArray');
  return isArray(array);
}

export function Add(a, b) {
  console.log('Add');
  return a + b
}
```

```
这个`array`函数未被使用，但是lodash-es这个包的部分代码还是会被build到bundle.js中
```

可以使用这个插件[webpack-deep-scope-analysis-plugin](https://github.com/vincentdchan/webpack-deep-scope-analysis-plugin)解决

## 参考

https://juejin.cn/post/6844903544756109319#heading-0
https://juejin.cn/post/6844903774192926728
https://juejin.cn/post/6844903687412776974#heading-8

