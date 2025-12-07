---
title: 【源码解读】p-limit 限制并发数
date: 2023-04-19 00:53:37
tags:
  - js
categories:
  - 源码解读
---

> github: https://github.com/sindresorhus/p-limit

## 使用方式

```js
import pLimit from 'p-limit';

const limit = pLimit(1); // 只能有一个promise在执行

const input = [
	limit(() => fetchSomething('foo')),
	limit(() => fetchSomething('bar')),
	limit(() => doSomething())
];

const result = await Promise.all(input);
console.log(result);
```

<!--more-->

## 函数源码

### 初始化（pLimit）

```js
import Queue from "yocto-queue";

export default function pLimit(concurrency) {
	const queue = new Queue();
	let activeCount = 0;

	const enqueue = (fn, resolve, args) => {
		queue.enqueue(run.bind(undefined, fn, resolve, args));

		(async () => {
             // 思考里讲解为什么这里需要Promise.resolve()
			await Promise.resolve()

			if (activeCount < concurrency && queue.size > 0) {
				queue.dequeue()();
			}
		})();
	};

	const generator = (fn, ...args) =>
		new Promise((resolve) => {
			enqueue(fn, resolve, args);
		});

	return generator;
}
```

`pLimit`函数的入参 concurrency 是最大并发数，调用一次 pLimit 会生成一个限制并发的函数 generator

依赖`yocto-queue`的队列能力，每次调用`generator`会返回个`promise`

1. 会向队列入队一个`run`（执行函数）
2. 如果当前在执行`promise`数量小于`concurrency`（并发数），就出队并执行

### 执行函数（run）

```js
const next = () => {
    activeCount--;

    if (queue.size > 0) {
        queue.dequeue()();
    }
};

const run = async (fn, resolve, args) => {
    activeCount++;

    const result = (async () => fn(...args))();

    resolve(result);

    try {
        await result;
    } catch {}

    next();
};
```

run函数执行

1. `activeCount`加一
2. 执行异步函数`fn`，并将结果传递给`resolve`
3. `await`了`result`，使得`next`执行有序
4. 执行`next`时表示`promise`结果已经返回，`activeCount`-1，并开始执行下一个`promise`

## 思考

### 为什么使用队列而不是数组

相关issus：[Improve performance](https://github.com/sindresorhus/p-limit/pull/47)

`shift`方法每次调用时, 都需要遍历一次数组, 将数组进行一次平移, 时间复杂度是O(n)
队列的`dequeue`时间复杂度则是O(1)

### 为什么在入队并且执行的时候，判断执行前需要await Promise.resolve()

相关issus：[Always run limited functions asynchronously](https://github.com/sindresorhus/p-limit/pull/28)

不加的话，有时候执行是同步的，有时候执行是异步的，有可能会导致在下一行代码执行之前状态就已经改变了，让程序运行结果不可预测

```js
(async () => {
    await Promise.resolve()

    if (activeCount < concurrency && queue.size > 0) {
        queue.dequeue()();
    }
})();
```

加上可以保证所有出队执行都是异步的

### 如何添加超时逻辑

```js
let timer = null;
const timerPromise = new Promise((resolve, reject) => {
  timer = setTimeout(() => {
    reject('time out');
  }, 1000);
});


Promise.all([
  timerPromise,
  fetchPromise,
])
.then(res => clearTimeout(timer))
.catch(err => console.error(err));
```

更正规的写法可以参考：[p-timeout](https://github.com/sindresorhus/p-timeout)

## 参考文章

[Node.js 并发能力总结](https://mp.weixin.qq.com/s/6LsPMIHdIOw3KO6F2sgRXg)