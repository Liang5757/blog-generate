---
title: 减少使用setInterval
date: 2021-01-22 22:25:34
tags:
  - js
categories:
  - 前端
---
`setInterval`很少使用在生成环境下，因为有如下几个缺点

## 缺点一：setInterval无视代码错误

`setInterval`执行的代码即使代码报错，它还会持续不断（不管不顾）地调用该代码

## 缺点二：setInterval无视网络延迟

假设你每隔一段时间就通过Ajax轮询一次服务器，看看有没有新数据（注意：如果你真的这么做了，那恐怕你做错了；建议使用[“补偿性轮询”（backoff polling）](https://link.juejin.im/?target=http%3A%2F%2Fgithub.com%2Fblog%2F467-smart-js-polling)）。而由于某些原因（服务器过载、临时断网、流量剧增、用户带宽受限，等等），你的请求要花的时间远比你想象的要长。但`setInterval`不在乎。它仍然会按定时持续不断地触发请求，最终你的客户端网络队列会塞满Ajax调用。

<!--more-->

## 缺点三：setInterval不保证按时执行

### 原因一：执行时间被前面任务阻塞

`setInterval`只是按时把时间推送到任务队列，因为js是单线程，所以如果前面的任务耗时较久，那么定时器的执行时间和我们预定的时间不一致

### 原因二：推送被忽略

`setInterval`在推送到任务队列前，**会检查上一次的任务是否仍在队列中**，如果**有则不填加**，`setTimeout`则会直接添加

## 用setTimeout模拟setInterval

```js
function interval(func, wait) {
  let timer = null;
  function interfunc() {
    func.call(null);
    timer = setTimeout(interfunc, wait);
  };
  timer = setTimeout(interfunc, wait);

  return timer;
}
```

