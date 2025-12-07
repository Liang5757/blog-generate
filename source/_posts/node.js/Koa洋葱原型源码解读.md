---
title: Koa洋葱原型源码解读
date: 2021-02-28 00:25:27
tags:
  - node.js
categories:
  - 后端
  - 源码解读
---
> 版本：2.13.1

## application.js

### 构造函数

```js
constructor(options) {
    ...
    this.middleware = [];
    ...
}
```

有一个`middleware`存函数
<!--more-->
### use方法

```js
use(fn) {
    ...
    this.middleware.push(fn);
    return this;
}
```

每次使用app.use()就会吧回调函数push到`middleware`里

### listen方法

```js
listen(...args) {
    const server = http.createServer(this.callback());
    return server.listen(...args);
}
```

创建一个服务，执行callback方法

### callback和handleRequest方法

```js
callback() {
    const fn = compose(this.middleware);

    if (!this.listenerCount('error')) this.on('error', this.onerror);

    const handleRequest = (req, res) => {
        const ctx = this.createContext(req, res); // node 原生的 req、res 对象把其中的属性挂载到 ctx 上
        return this.handleRequest(ctx, fn); // 调用
    };

    return handleRequest;
}

handleRequest(ctx, fnMiddleware) {
    const res = ctx.res;
    const onerror = err => ctx.onerror(err);
    // 处理响应
    const handleResponse = () => respond(ctx);
    return fnMiddleware(ctx).then(handleResponse).catch(onerror);
}
```

compse函数返回一个中间件函数，在handleRequest执行中间件函数，如果全部 resolve 了就可以调用 handleResponse 发送给客户端

本片博客的重点就是compose是怎么实现koa的洋葱模型的

## koa-compose.js

```js
app.use(async (ctx, next) => {
    console.log("1");
    await next();
    console.log("4");
});

app.use(async (ctx, next) => {
    console.log("2");
    await next();
    console.log("3");
});
```

### compose函数

```js
function compose (middleware) {
    ...
    /**
   * @param {Object} context
   * @return {Promise}
   * @api public
   */
    return function (context, next) {
        // last called middleware #
        let index = -1
        return dispatch(0)

        function dispatch (i) {
            if (i <= index)
                return Promise.reject(new Error('next() called multiple times'))
            index = i
            let fn = middleware[i]
            if (i === middleware.length) fn = next
            if (!fn) return Promise.resolve()
            try {
                return Promise.resolve(fn(context, dispatch.bind(null, i + 1)));
            } catch (err) {
                return Promise.reject(err)
            }
        }
    }
}
```

- `compose`函数接收`middleware`数组，`dispatch(0)`即开始分发一号中间件。

- `dispatch(0)`内部，此时 fn 为一号中间件，会走到 try/catch 块，尝试执行`Promise.resolve(fn(context, dispatch.bind(null, i + 1)))`，即一号中间件此时获得入参`context`、`dispatch(1)`。

- 一号中间件开始执行，遇到 next()（即dispatch(1)），控制权移交，执行 dispatch(1)，此时二号中间件获得入参`context`、`dispatch(2)`。

- 二号中间件开始执行，执行到`await next()`时，再重复上述逻辑，dispatch(2)，但是这一次会停在这里：

	```js
	let fn = middleware[i];
	if (i === middleware.length) fn = next;
	if (!fn) return Promise.resolve();
	```

	fn = next，这里的 next 由于并没有值，所以会直接 return 一个立即 resolve 的 Promise。也就是说二号中间件内部的 await next()会立刻返回。

- 二号中间件做完自己的事后，相当于一号中间件内部的`await next()`返回了，因此控制权就归还给一号中间件。

## 如果中间件中的`next()`方法报错了怎么办。

```js
ctx.onerror = function {
    this.app.emit('error', err, this);
};

listen(){
    const  fnMiddleware = compose(this.middleware);
    if (!this.listenerCount('error')) this.on('error', this.onerror);
    const onerror = err => ctx.onerror(err);
    fnMiddleware(ctx).then(handleResponse).catch(onerror);
}

onerror(err) {
    // 代码省略
    // ...
}
```

答：中间件链错误会由`ctx.onerror`捕获，该函数中会调用`this.app.emit('error', err, this)`（因为`koa`继承自`Emitter`，所以有`emit`和`on`等方法），可以使用`app.on('error', (err) => {})`，或者`app.onerror = (err) => {}`进行捕获。

## 参考文章

https://juejin.cn/post/6844904088220467213#heading-16
https://linbudu.top/posts/2020/02/25/koa%E6%BA%90%E7%A0%81%E7%B2%BE%E8%AF%BB.html#new-%E4%B8%80%E4%B8%AA-koa-%EF%BC%8C%E5%8F%91%E7%94%9F%E4%BA%86%E4%BB%80%E4%B9%88%EF%BC%9F

