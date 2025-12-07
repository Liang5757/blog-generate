---
title: 事件 Event
date: 2020-07-31 17:13:17
tags:
  - js
categories:
  - 前端
---

**事件**是您在编程时系统内发生的动作或者发生的事情——系统会在事件出现的时候触发某种信号并且会提供一个自动加载某种动作，列举一些可能发生的事件。

- 用户在某个元素上点击鼠标或悬停光标。
- 用户在键盘中按下某个按键。
- 用户调整浏览器的大小或者关闭浏览器窗口。
- 一个网页停止加载。
- 提交表单。
- 播放、暂停、关闭视频。
- 发生错误。

<!--more-->

### 1.事件处理器

可以为每一个事件绑定一个**事件处理器**(**事件监听器**)，用以事件被激发时做出的回应，有三种事件处理器。

#### 1.1 事件处理器属性（DOM0级）

```javascript
var btn = document.querySelector('button');
btn.onclick = function(event){alert('Hello world');};
```

该函数在定义时，可以传入一个 `event` 形式的参数，该方式的问题在于一次只能绑定一个。

除了onclick还有一些常见的事件处理器属性

| 事件处理器属性    | 备注                                                     |
| :---------------- | :------------------------------------------------------- |
| btn.onfocus       | 当前元素获得键盘焦点时会触发`focus`事件                  |
| btn.onblur        | 当一个元素失去焦点时会触发`blur`事件                     |
| btn.ondblclick    | 当前元素上双击鼠标左键会触发`dblclick`事件               |
| window.onkeypress | 当用户在键盘上按下任意键时，**应当**会触发 keypress 事件 |
| window.onkeydown  | 当用户按下键盘上的按键时会触发`keydown`事件              |
| window.onkeyup    | 在当前元素上释放键盘按键时会触发`keyup事件`              |
| btn.onmouseover   | 当鼠标移动到一个元素上时,会在这个元素上触发mouseover事件 |
| btn.onmouseout    | 当鼠标离开一个元素时,会在这个元素上触发mouseout事件      |

#### 1.2 行内事件处理器 - 请勿使用

```javascript
<button onclick="alert('Hello World');">Press me</button>
```

我们应该避免使用这种方式。因为它会使标记数量变大，而可读性却较差。 内容/结构 和 行为之间没有很好的分离，使得在处理bug时非常困难。

#### 1.3 addEventListener()（DOM2级）

```javascript
var btn = document.querySelector('button');
btn.addEventListener('click', function(){alert('Hello world');}, false);
```

##### addEventListener()参数

| 参数名              | 传入参数                                                     |
| ------------------- | ------------------------------------------------------------ |
| **type**            | 表示监听[事件类型](https://developer.mozilla.org/zh-CN/docs/Web/Events)的字符串。 |
| **listener**        | 实现了Event接口的对象或者函数                                |
| **options** 可选    | 一个指定有关 `listener `属性的可选参数**对象**               |
| **useCapture** 可选 | Boolean，默认为false，在事件冒泡阶段调用事件处理程序，反之为捕获 |

##### addEventListener()返回值

undefined

在DOM2级可以给同一个监听器注册多个处理器，而DOM0级不能实现这一点：

```js
myElement.onclick = functionA;
myElement.onclick = functionB;
```

第二行会覆盖第一行，但是下面这种方式就会正常工作了：

```js
myElement.addEventListener('click', functionA);
myElement.addEventListener('click', functionB);
```

当元素被点击时两个函数都会工作

**注意**：通过addEventListener()添加的事件处理程序只能用removeEventListener()来移除，并且移除时传入的参数必须与添加时传入的参数一样。

#### 1.4 IE事件处理器

IE用了attachEvent()，detachEvent()，接收两个参数，**事件名称、事件处理程序**，通过 attachEvent() 添加的事件处理程序都会被添加到冒泡阶段,所以平时为了兼容更多的浏览器最好将事件添加到事件冒泡阶段,IE8及以前只支持事件冒泡;

例子：

```javascript
var btn = document.getElementById('btn');
var handlers = function(){
	console.log(this === window);  //true,注意attachEvent()添加的事件处理程序运行在全局作用域中;
};
btn.attachEvent('onclick', handlers);
```

##### 跨浏览器事件处理程序

```javascript
//创建的方法是addHandlers(),removeHandlers(),这两个方法属于一个叫EventUtil的对象;但是这个没有考虑到IE中作用域的问题，不过就添加和移除事件还是足够的。
 
var EventUtil = {
   addHandlers: function (element, type, handlers) {
      if (element.addEventListener) {
         element.addEventListener(type, handlers, false);
      } else if (element.attachEvent) {
         element.attachEvent(on + type, handlers);
      } else {
         element['on' + type] = handlers;
      }
   },
   removeHandlers: function (element, type, handlers) {
      if (element.removeEventListener) {
         element.removeEventListener(type, handlers, false);
      } else if (element.detachEvent) {
         element.detachEvent(on + type, handlers);
      } else {
         element['on' + type] = null;
      }
   }
};
```

例子：

```javascript
var btn = document.getElementById('btn');
var handlers = function() {
   console.log('123')
};
EventUtil.addHandlers(btn, 'click', handlers);
EventUtil.removeHandlers(btn, 'click', handlers);
```

在同一个对象上注册事件，并不一定按照注册顺序执行，冒泡或捕获模式会影响其被触发的顺序

### 2.事件冒泡及捕获

当一个事件发生在具有父元素的元素上，现代浏览器运行两个不同的阶段——捕获阶段和冒泡阶段。

#### 2.1捕获阶段

- 浏览器检查元素的最外层祖先`<html>`，是否在捕获阶段中注册了一个`onclick`事件处理程序，如果是，则运行它。
- 然后，它移动到`<html>`中单击元素的下一个祖先元素，并执行相同的操作，然后是单击元素再下一个祖先元素，依此类推，直到到达实际点击的元素。

#### 2.2冒泡阶段

- 浏览器检查实际点击的元素是否在冒泡阶段中注册了一个`onclick`事件处理程序，如果是，则运行它
- 然后它移动到下一个直接的祖先元素，并做同样的事情，然后是下一个，等等，直到它到达`<html>`元素。

#### 2.3用stopPropagation()修复问题

当在事件对象上调用该函数时，它只会让当前事件处理程序运行，但事件不会在**冒泡**链上进一步扩大，因此将不会有更多事件处理器被运行(不会向上冒泡)。

### 3.事件对象 event

`event`，`evt`或简单的`e`。 这被称为**事件对象**，它被自动传递给事件处理函数，以提供额外的功能和信息，比如： 

```javascript
function bgChange(e) {
  var rndCol = 'rgb(' + random(255) + ',' + random(255) + ',' + random(255) + ')';
  e.target.style.backgroundColor = rndCol;
  console.log(e);
}  
btn.addEventListener('click', bgChange);
```

事件对象 `e` 的`target`属性始终是事件刚刚发生的元素的引用。

#### 跨浏览器的事件对象

虽然DOM和IE中对象不同，但是两者event中的全部信息和方法都是类似的只是实现方式不同，可以用前面提到过的EventUtil对象来求同存异。

```javascript
var EventUtil = {
    addHandler: function (element, type, handler) {
        if (element.addEventListener) {
            element.addEventListener(type, handler, false);
        } else if (element.attachEvent) {
            element.attachEvent(on + type, handler);
        } else {
            element['on' + type] = handler;
        }
    },

    getEvent: function (event) {
        return event ? event : window.event;

    },

    getTarget: function (event) {
        return event.target || event.srcElement;
    },

    preventDefault: function (event) {
        if (event.preventDefault) {
            event.preventDefault();
        } else {
            event.returnValue = false;
        }
    },

    stopPropagation: function (event) {
        if (event.stopPropagation) {
            event.stopPropagation();
        } else {
            event.cancelBubble = true;
        }
    },

    removeHandler: function (element, type, handler) {
        if (element.removeEventListener) {
            element.removeEventListener(type, handler, false);
        } else if (element.detachEvent) {
            element.detachEvent(on + type, handler);
        } else {
            element['on' + type] = null
        }
    }
};
```

### 4.事件委托

#### 4.1 问题初现

给100个按钮绑定事件，传统的做法就是：通过DOM操作获取每一个按钮元素分别绑定事件处理函数。

但在JavaScript中，添加到页面上的事件处理程序数量将直接关系到页面的整体运行性能，因为需要不断的与dom节点进行交互，访问dom的次数越多，引起浏览器重绘与重排的次数也就越多，就会延长整个页面的交互就绪时间。

而每一个函数都是一个对象，所以都会占用内存，内存占用的越多性能毋庸置疑的会变得越差。

#### 4.2 解决方案——事件委托

事件委托就是利用事件冒泡，即：利用冒泡机制将一类事件触发尽可能高的委托给其父节点或祖先节点来触发事件处理函数，这样只需要定义一个函数，访问一次DOM对象，减少了内存的占用以及访问DOM元素的时间，降低了性能的消耗。

**例子**：
点击某一个 Li 标签时，将 Li 的背景色显示在 P 标签内，并将 P 标签中的文字颜色设置成 Li 的背景色，下面是html

```html
<ul class="palette">
	<li style="background-color:crimson"></li>
	<li style="background-color:bisque"></li>
	<li style="background-color:blueviolet"></li>
	<li style="background-color:coral"></li>
	<li style="background-color:chartreuse"></li>
	<li style="background-color:darkolivegreen"></li>
	<li style="background-color:cyan"></li>
	<li style="background-color:#194738"></li>
</ul>

<p class="color-picker"></p>
```

**实现方法**

```javascript
const list = document.querySelector(".palette");
list.onclick = function(e) {
    e = e || window.event;
    const t = e.target || e.srcElement;
    const p = document.getElementsByClassName('color-picker')[0];

    if (t.nodeName.toLowerCase() === 'li') {
        p.innerHTML = t.style.backgroundColor;
        p.style.color = t.style.backgroundColor;
    }
}
```

适合事件委托的事件有：click，mousedown，mouseup，keydown，keyup，keypress。

**参考文章**
https://www.cnblogs.com/lazychen/p/5664788.html
