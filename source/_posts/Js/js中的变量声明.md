---
title: js中的变量声明
date: 2020-07-31 17:14:10
tags:
  - js
categories:
  - 前端
---
## 一、 es5和es6中变量声明的区别

|            | 变量提升 | 块级作用域 | 重复声明 |
| ---------- | -------- | ---------- | -------- |
| var        | 会       | 无         | 能       |
| let、const | 不会     | 有         | 不能     |

<!--more-->

## 二、各种变量声明的特点

### 1. var

当var被作用于全局作用域时，他会把变量作为**全局对象（浏览器环境中的window对象）的属性**

由于没有块级作用域，所以任何位置的变量的声明在**js引擎扫描代码**的时候，将其**提升至所在作用域的顶部**

```javascript
function getValue(condition) {
	if (condition) {
		var value = "blue";
		return value;
	} else {
		// 此处可访问变量value，其值为undefined
		return null;
	}
}
```

被js引擎解析成如下所示

```js
function getValue(condition) {
    var value;		// value的生命提升到所在作用域的顶部
    
	if (condition) {
		value = "blue";
		return value;
	} else {
		// 此处可访问变量value，其值为undefined
		return null;
	}
}
```

### 2. let/const的共同特点

#### 2.1 具有块级作用域

存在于

- 函数内部
- 块中（字符 { 和 } 之间的区域）

##### 循环中的块级作用域绑定

var声明使得变量到循环外仍能访问

```js
var funcs = [];

for (var i = 0; i < 10; i++) {
	funcs.push(function() {
		console.log(i);
	});
}

funcs.forEach(function(func) {
	func();		// 输出10次数字10
});			
```

为了解决这个问题，可以使用**立即执行函数（IIFE）**，**为接受的每一个参数i都创建了一个副本并储存为变量value**

```js
// 循环修改成
for (var i = 0; i < 10; i++) {
	funcs.push((function(value) {
		return function() {
            console.log(value);
        }
	}(i)));
}
```

而在es6中的**let**简化了这个过程，**每次迭代循环都会创建一个新变量，并以之前迭代中同名变量的值将其初始化**

**特别的**如果在for循环中用**const替换了let**，则**const声明的变量不能在后续的循环中被修改**，而在**for-in或for-of循环中使用const时的行为与使用let一致**，因为每次迭代是**创建一个新绑定**

#### **2.2 禁止重复声明**

​	会报 **Identifier '变量' has already been declared** 的错

#### 2.3 暂时性死区（Temporal Dead Zone）

js引擎在扫描代码的时候发现**let**或者**const**声明的变量，会将他**放到TDZ**，访问TDZ中的变量会触发运行时错误，**只有执行过变量声明语句后，才会把变量从TDZ中移除**

由于块级作用域，**所以在声明的变量的作用域外使用则不会报错**

```js
console.log(typeof value);		// undefined

if (condition) {
   	let value = "blue";
}
```

## 三、函数声明

```js
// 函数表达式
var f = function() {
      console.log(1);  
}

// 直接声明
function f (){
     console.log(2);
}

// 如果两种声明方式声明同一个函数名，最终执行的是 函数表达式
f();	// 1
```

1. 第一种方式：函数只能在声明之后调用，因为这种函数声明是在函数运行的阶段才赋值给变量 f 的
2. 第二种方式：函数可以在声明函数的作用域内任一地方调用。因为这种方式，是在函数解析阶段赋值给标识符 f .

**函数声明提升优先于变量提升，且不会被变量声明覆盖，但是会被变量赋值覆盖**

```js
console.log(foo);	// [Function: foo]
function foo(){
	console.log("函数变量");
}
var foo = "变量";
console.log(typeof foo)		//string
```

下面来看一段代码

```js
{
    function test(a) {
        console.log(typeof test); // function
        test = a;	// 1
    }
    test(1);
    console.log(typeof test);	// number
}
console.log(typeof test);	// function
```

由于函数提升，会提升到块级作用域外，所以输出外层的test的类型为function，由于块级作用域，**变量重写只能在块内生效**

## 2021.2.10 块级作用域以及TDZ的新理解

代码执行的第一步是编译并创建执行上下文，如下图所示，`var`声明的变量在变量环境中，`let`和`const`在词法环境中，词法环境其实 是维护了一个小型栈结构，**栈底是最外层的变量**

![image-20210210235407436](image-20210210235407436.png)

第二步再执行代码，当作用域执行完成后，该作用域的信息就会从词法环境中弹出。

而变量经历了 **创建、初始化、赋值**，创建和初始化是在编译阶段完成的，变量就会放置到对应的环境，但是**初始化要看变量的声明方式**。

- var的创建和初始化被提升，赋值不会被提升
- let、const的创建被提升，初始化和赋值不会被提升
- function的创建、初始化和赋值均会被提升

我们看如下的代码

```js
let myname = 'liang1'
{
  console.log(myname)
  let myname = 'liang2' // 在该块上部形成了暂时性死区
}
```

代码出现错误`ReferenceError: Cannot access 'myname' before initialization`，因为在块内访问myname的时候，在词法环境内已经有该块内创建的myname变量，在初始化前访问就报错了。

**参考文章**
https://zhuanlan.zhihu.com/p/126237126
https://www.cnblogs.com/miacara94/p/9173843.html
《深入理解ES6》

1. 第一种方式：函数只能在声明之后调用，因为这种函数声明是在函数运行的阶段才赋值给变量 f 的
2. 第二种方式：函数可以在声明函数的作用域内任一地方调用。因为这种方式，是在函数解析阶段赋值给标识符 f .

**函数声明提升优先于变量提升，且不会被变量声明覆盖，但是会被变量赋值覆盖**

```js
console.log(foo);	// [Function: foo]
function foo(){
	console.log("函数变量");
}
var foo = "变量";
console.log(typeof foo)		//string
```

下面来看一段代码

```js
{
    function test(a) {
        test = a;	// 1
    }
    test(1);
    console.log(typeof test);	// number
}
console.log(typeof test);	// function
```

由于函数提升，会提升到块级作用域外，所以输出外层的test的类型为function，由于块级作用域，**变量重写只能在块内生效**

**参考文章**
https://zhuanlan.zhihu.com/p/126237126
https://www.cnblogs.com/miacara94/p/9173843.html
《深入理解ES6》
