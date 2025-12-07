---
title: 【源码解读】crossvent源码解读
date: 2021-04-06 23:46:07
tags:
  - js
categories:
  - 源码解读
---
> 其实就是一个封装事件绑定的库，但是看到了一些技巧记录下来
> github地址：https://github.com/bevacqua/crossvent/blob/master/src/crossvent.js

## 从出口开始

```javascript
module.exports = {
  add: addEvent,
  remove: removeEvent,
  fabricate: fabricateEvent
};
```

<!--more-->

## add和remove函数

`addEvent`实际上就是判断window是否有`addEventListener`，如果没有就用`attachEvent`，remove同理

```js
var addEvent = addEventEasy;
var removeEvent = removeEventEasy;

if (!global.addEventListener) {
  addEvent = addEventHard;
  removeEvent = removeEventHard;
}

function addEventEasy (el, type, fn, capturing) {
  return el.addEventListener(type, fn, capturing); // capturing如果为true则在事件捕获阶段执行，默认为false
}

function addEventHard (el, type, fn) {
  return el.attachEvent('on' + type, wrap(el, type, fn));
}

function removeEventEasy (el, type, fn, capturing) {
  return el.removeEventListener(type, fn, capturing);
}

function removeEventHard (el, type, fn) {
  var listener = unwrap(el, type, fn);
  if (listener) {
    return el.detachEvent('on' + type, listener);
  }
}
```

这里需要注意：在不支持`addEventListener`的时候，使用`attachEvent`和`detachEvent`事件回调函数多做了一层`wrap`和`unwrap`处理

## wrap和unwrap

```js
var hardCache = [];

function wrap (el, type, fn) {
  // 如果已经注册过这个type的事件，则unwrap即从hardCache删除事件，并在下面重新添加到hardCache数组里
  var wrapper = unwrap(el, type, fn) || wrapperFactory(el, type, fn); 
  hardCache.push({
    wrapper: wrapper,
    element: el,
    type: type,
    fn: fn
  });
  return wrapper;
}

function unwrap (el, type, fn) {
  var i = find(el, type, fn);
  if (i) {
    var wrapper = hardCache[i].wrapper;
    hardCache.splice(i, 1); // free up a tad of memory
    return wrapper;
  }
}

function find (el, type, fn) {
  var i, item;
  for (i = 0; i < hardCache.length; i++) {
    item = hardCache[i];
    if (item.element === el && item.type === type && item.fn === fn) {
      return i;
    }
  }
}
```

可以看到是利用一个数组存储事件回调，在绑定事件和解绑事件同时对hardCache数组进行操作，但是为啥要这样呢。可以看到在wrap函数的第一行`var wrapper = unwrap(el, type, fn) || wrapperFactory(el, type, fn);` ，实际上就是如果之前没有绑定这个type的事件，就调用`wrapperFactory`对`fn`进行二次封装，**`wrapperFactory`这个封装主要是为了解决event参数内的属性的兼容性**。

这就解释了刚才提出的疑问，就是因为多了这层解决浏览器兼容性的包装，在解绑事件的时候，如果只是传递原来的fn，则不能解绑事件，需要一个hardCache数组来存取事件回调。

## wrapperFactory

```js
function wrapperFactory (el, type, fn) {
  return function wrapper (originalEvent) {
    var e = originalEvent || global.event;
    e.target = e.target || e.srcElement;
    e.preventDefault = e.preventDefault || function preventDefault () { e.returnValue = false; };
    e.stopPropagation = e.stopPropagation || function stopPropagation () { e.cancelBubble = true; };
    e.which = e.which || e.keyCode;
    fn.call(el, e);
  };
}
```

这层封装可以使event的各个属性在不同浏览器下行为一致。

add和revome就讲完了，我们来看看导出的最后一个函数fabricate

## fabricate

```js
function fabricateEvent (el, type, model) {
  var e = eventmap.indexOf(type) === -1 ? makeCustomEvent() : makeClassicEvent(); // 判断事件类型 并 创建
  if (el.dispatchEvent) { // 触发事件
    el.dispatchEvent(e);
  } else {
    el.fireEvent('on' + type, e);
  }
  function makeClassicEvent () { // 创建原生事件
    var e;
    if (doc.createEvent) {
      e = doc.createEvent('Event');
      e.initEvent(type, true, true);
    } else if (doc.createEventObject) {
      e = doc.createEventObject();
    }
    return e;
  }
  function makeCustomEvent () { // 创建自定义事件
    return new customEvent(type, { detail: model });
  }
}
```

`eventmap`是原生事件名的集合，如果没有在`eventmap`找到则创建一个自定义事件，否则创建一个原生事件，然后触发事件。

那么eventmap是怎么生成的呢，我们下面来看看

## 获取原生事件名集合

```js
var eventmap = [];
var eventname = '';
var ron = /^on/;

for (eventname in global) {
  if (ron.test(eventname)) {
    eventmap.push(eventname.slice(2));
  }
}

module.exports = eventmap;
```

其实就是正则匹配window上的开头为on的属性名

## 总结

看到了挺多浏览器兼容的处理方式，以及原生事件名获取的骚操作，不过仍有一些兼容方式没看懂是兼容哪些浏览器的。
