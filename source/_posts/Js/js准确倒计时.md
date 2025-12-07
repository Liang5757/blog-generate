---
title: js准确倒计时
date: 2021-05-25 00:03:51
tags:
  - js
categories:
  - 前端
---
## 引入

如果用最初始的`setTimeout`递归实现定时器，一秒执行一次回调，则代码如下

<!--more-->

```js
// 模拟执行大量代码
setInterval(() => {
  let i = 0;
  while (i++ < 100000000) {
  }
}, 0);

function countDown (fn, time) {
  let startTime = new Date().getTime(), count = 0, second = 1000;
  let timeCounter = null;
  
  return function interFunc () {
    let offset = new Date().getTime() - (startTime + count * second);
    console.log("误差：" + offset + "ms，下一次执行：" + 1000 + "ms后，离活动开始还有：" + time + "ms");
    time -= second;
    count++;
    if (time < 0) {
      clearTimeout(timeCounter);
    } else {
      timeCounter = setTimeout(() => {
        fn.apply(this, arguments);
        interFunc();
      }, 1000);
    }
  }
}

countDown(function () {}, 10000)(); // 测试代码
```

![image-20210524235731030](image-20210524235731030.png)

可以看到误差是会越来越多的。

## 解决

核心思想就是通过计算 **当前时间 - 应该到的时间** 计算出**时间偏移量**，**下一次延迟时间就是 1s - 偏移量**

```js
// 模拟执行大量代码
setInterval(() => {
  let i = 0;
  while (i++ < 100000000) {
  }
}, 0);

// 倒计时
function countDown (fn, time) {
  let startTime = new Date().getTime(), count = 0, second = 1000;
  let timeCounter = null;

  return function interFunc () {
    let offset = new Date().getTime() - (startTime + count * second);
    let nextTime = second - offset;
    if (nextTime < 0) {
      nextTime = 0
    }
    console.log("误差：" + offset + "ms，下一次执行：" + nextTime + "ms后，离活动开始还有：" + time + "ms");
    time -= second;
    count++;
    if (time < 0) {
      clearTimeout(timeCounter);
    } else {
      timeCounter = setTimeout(() => {
        fn.apply(this, arguments);
        interFunc();
      }, nextTime);
    }
  }
}

countDown(function () {}, 10000)(); // 测试代码
```

![image-20210525000118726](image-20210525000118726.png)

可以看到虽然仍有误差，但不会随着时间增大
