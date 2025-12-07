---
title: CJS、ESM、UMD模块化标准
date: 2021-01-19 23:27:09
tags:
  - js
categories:
  - 前端
---
## 1. CommonJS

> node.js的实现中采用了CommonJS标准的一部分，并在其基础上进行了一些调整

### 使用方式

使用`require`和`exports`或者`module.exports`进行模块的导入导出

```js
// node.js为了简化操作，有var exports = module.exports
// 两者一致，那就说明，我可以使用任意一方来导出内部成员
console.log(exports === module.exports) // true

// 导出的对象近似于此种形式，exports相当于module.exports的引用
// var module = {
//   exports: {
//     foo: 'bar',
//   }
// }
```

<!--more-->

### 特点

1.CJS 模块输出的是值的拷贝

2.一个文件就是一个模块，name.js中的作用域是影响不到index.js的。

```js
// name.js
let name = 'name.js';

// index.js
let name = 'index.js';
require('./name');
console.log(name); // index.js
```

原因

```js
// 当node在执行模块中的代码时，它会首先在代码的最顶部，添加如下代码
function (exports, require, module, __filename, __dirname) {
// 在代码的最底部，，添加如下代码
}
// 所以可以理解一个文件就是一个模块
```

3.以同步的方式加载模块，模块加载顺序即在代码中的顺序，每个require语句会短暂阻塞代码的运行，知道模块加载完毕。不过这个加载不是通过网络加载，而是从内存或者文件系统中加载，所以这个过程很快。也导致CJS不适合浏览器

4.有缓存机制，已经被引入过的模块，不会再一次引入

```js
// 由使用方式小节可知，其实模块会有一个module对象，而这个module对象会存放模块的信息
// 其中有一个属性loaded用于记录该模块是否被加载过，默认为false，第一次加载后设为true，之后如果该属性为true，则不会再次执行改模块的代码

// cache.js
let a = 1;
console.log(a++);

// index.js
require('./cache'); // 输出1
require('./cache'); // 由于缓存机制，无输出
```

5.循环引用下的行为

将上文练习的b.js文件修改成下面这样：

```javascript
const a = require('./a.js');
console.log(a, 'b中拿到的a');

module.exports = {
  b1: '111',
  b2: '222',
  b3: '333',
  ba: a,
};
```

a.js文件内容如下:

```javascript
const b = require('./b.js');
console.log(b, 'a中拿到的b');
```

执行`node a.js`，打印出

```javascript
{} b中拿到的a
{ b1: '111', b2: '222', b3: '333', ba: {} } a中拿到的b
```

首先执行a.js，a中引用了b，所以b开始执行，b中又引用了a，此时a没有任何导出内容，所以b拿到的a是一个空对象。

修改a.js文件内容如下：

```javascript
exports.a1 = '111';

const b = require('./b.js');
console.log(b, 'a中拿到的b');

exports.a2 = '222';
```

可以看见在导入b之前先导出了a1，执行`node a.js`，打印的内容是：

```javascript
{ a1: '111' } b中拿到的a
{ b1: '111', b2: '222', b3: '333', ba: { a1: '111' } } a中拿到的b
```

说明在b中导入a的时候的拿到的是那个时刻a中已经导出的内容，如果没有导出，就会拿到一个空对象。

exports 是动态执行的，具体 require 能获取到的值，取决于模块的运行情况

## 2.AMD

> Asynchronous Module Definition（异步模块规范），最老的方式之一，专为浏览器而设计，[RequireJS](https://requirejs.org/docs/release/2.3.6/minified/require.js)实现了AMD API

### 使用方式

用`define`方法定义模块，用`require`导出模块

```js
define(
    '模块名',
    ['依赖数组']，
    function([依赖数组]){     //工厂函数
        ...
    }
);
    
require(['jquery']，
    function($){        //回调函数
    ...
    }
);
```

### 特点

1. 可以定义具名模块，也可以定义匿名模块。具名模块通过开发者定义的名字加载，匿名模块隐式的以文件名加载
2. require调用会发送一个请求来下载模块
3. 异步加载，只有在回调函数里才能获取新拿到的API
4. 可以在代码的任何地方使用require加载另一个模块

## 3. ESM

> 与CJS和AMD不同，es6的模块是静态的，

### 使用方式

用`import`和`import()`进行导入，用`export`、`export default`进行导出

### 特点

1.ES6 模块输出的是值的引用

```javascript
// lib.js
export let counter = 3;
export function incCounter() {
  counter++;
}

// main.js
import { counter, incCounter } from './lib';
console.log(counter); // 3
incCounter();
console.log(counter); // 4
```

2.会动态的从被加载的模块中取值

```javascript
// m1.js
export var foo = 'bar';
setTimeout(() => foo = 'baz', 500);

// m2.js
import {foo} from './m1.js';
console.log(foo); // bar
setTimeout(() => console.log(foo), 500); // baz
```

3.导入的模块变量是**只读**的，如果修改就会报错

```javascript
// lib.js
export let obj = {};

// main.js
import { obj } from './lib';
obj.prop = 123; // OK
obj = {}; // TypeError
```

4.循环引用下的行为

```js
// a.mjs
import {bar} from './b';
console.log('a.mjs');
console.log(bar);
export let foo = 'foo';

// b.mjs
import {foo} from './a';
console.log('b.mjs');
console.log(foo); // ReferenceError: foo is not defined
export let bar = 'bar';
```

执行`a.mjs`，在`a.mjs`中引擎发现它加载了`b.mjs`，因此会优先执行`b.mjs`，在`b.mjs`中发现它又加载了`a.mjs`，形成循环引用，已知`a.mjs`输入了`foo`接口，这时不会去执行`a.mjs`，而是认为这个接口已经存在了，继续往下执行，执行到第三行`console.log(foo)`的时候，才发现这个接口根本没定义，因此报错。

与CJS不同的是，ESM可以利用函数提升来解决上述问题，在`b.mjs`运行的时候，已经有`foo`的定义了

让我们看另一个例子

```js
// a.mjs
export let a_done = false;
import { b_done } from './b';
console.log('a.js: b.done = %j', b_done);
console.log('a.js执行完毕');

// b.mjs
import { a_done } from './a';
console.log('b.js: a.done = %j', a_done); // 此处报错
export let b_done = true;
console.log('b.js执行完毕');
```

执行`a.mjs`，因为import具有提升效果，所以在`b.mjs`中的第二行就会报`ReferenceError: Cannot access 'a_done' before initialization`的错误，同样的例子CJS由于没有提升，则可以获取到`a_done`为false

ps：babel编译过后上述例子不会报错

## 4.UMD

> UMD：Universal Module Definition（通用模块规范）是由社区想出来的一种整合了CommonJS和AMD两个模块定义规范的方法。

UMD模块的顶端通常都会有如下的代码，用来判断模块加载器环境。

```js
(function (root, factory) {
    if (typeof define === 'function' && define.amd) {
        // AMD
        define(['jquery'], factory);
    } else if (typeof exports === 'object') {
        // CommonJS
        module.exports = factory(require('jquery'));
    } else {
        // 全局变量
        root.returnExports = factory(root.jQuery);
    }
}(this, function ($) {
    // ...
}));
```

## 参考

《深入浅出Webpack》
https://es6.ruanyifeng.com/#docs/module-loader
https://juejin.cn/post/6870141103958589454
https://juejin.cn/post/6844903861166014478

