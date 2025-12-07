---
title: 【源码解读】Axios源码解析
date: 2021-08-25 01:20:56
tags:
  - js
categories:
  - 源码解读
---
## 工具类

### bind

```js
function bind(fn, thisArg) {
  return function wrap() {
    var args = new Array(arguments.length);
    for (var i = 0; i < args.length; i++) {
      args[i] = arguments[i];
    }
    return fn.apply(thisArg, args);
  };
};
```
<!--more-->

### extend

```js
function extend(a, b, thisArg) {
  forEach(b, function assignValue(val, key) {
    if (thisArg && typeof val === 'function') {
      a[key] = bind(val, thisArg);
    } else {
      a[key] = val;
    }
  });
  return a;
}
```

### merge

```js
function merge(/* obj1, obj2, obj3, ... */) {
  var result = {};
  function assignValue(val, key) {
    if (isPlainObject(result[key]) && isPlainObject(val)) {
      result[key] = merge(result[key], val);
    } else if (isPlainObject(val)) {
      result[key] = merge({}, val);
    } else if (isArray(val)) {
      result[key] = val.slice();
    } else {
      result[key] = val;
    }
  }

  for (var i = 0, l = arguments.length; i < l; i++) {
    forEach(arguments[i], assignValue);
  }
  return result;
}
```

## 从使用方式看起

对几种调用方式做了编号

1. `axios(option)`
2. `axios(url[, option])`
3. `axios[method](url[, option])`（`get、delete`等方法）
4. `axios[method](url[, data[, option]])`（`post、put`等方法）
5. `axios.request(option)`

让我们看看源码他是怎么做到能够支持这么多调用方式的

```js

// /lib/axios.js
function createInstance(defaultConfig) {
  var context = new Axios(defaultConfig);

  // 等同于 Axios.prototype.request.bind(context)
  // request方法中对第一个参数是否为 string 做了判断，使支持1、2种调用方式
  var instance = bind(Axios.prototype.request, context);

  // 把Axios.prototype上的方法扩展到instance对象上，
  // 这样 instance 就有了 get、post、put、request等方法，使支持3、4、5调用方式
  // 并指定上下文为context，这样执行Axios原型链上的方法时，this会指向context
  utils.extend(instance, Axios.prototype, context);

  // 把context对象上的自身属性和方法扩展到instance上
  // 这样，instance 就有了 defaults、interceptors 属性。（这两个属性后面我们会介绍）
  utils.extend(instance, context);

  return instance;
}

// 接收默认配置项作为参数（后面会介绍配置项），创建一个Axios实例，最终会被作为对象导出
var axios = createInstance(defaults);
```

主要是为了能够做到1、2种调用方式，所以是返回request函数，并在其上挂载Axios实例的方法

接下来看看上述代码操作的主体：Axios、Axios.prototype.request 以及 Axios.prototype上的请求方法

```js
// /lib/core/Axios.js
function Axios(instanceConfig) {
  this.defaults = instanceConfig;
  this.interceptors = {
    request: new InterceptorManager(),
    response: new InterceptorManager()
  };
}
Axios.prototype.request = function request(config) {
  if (typeof config === 'string') { // 使支持1、2两种调用方式
    config = arguments[1] || {};
    config.url = arguments[0];
  } else {
    config = config || {};
  }
  // ...
};

// Provide aliases for supported request methods
utils.forEach(['delete', 'get', 'head', 'options'], function forEachMethodNoData(method) {
  Axios.prototype[method] = function(url, config) {
    return this.request(mergeConfig(config || {}, {
      method: method,
      url: url,
      data: (config || {}).data
    }));
  };
});

utils.forEach(['post', 'put', 'patch'], function forEachMethodWithData(method) {
  Axios.prototype[method] = function(url, data, config) {
    return this.request(mergeConfig(config || {}, {
      method: method,
      url: url,
      data: data
    }));
  };
});
```

结合上面`createInstance`的代码，我们可以看到为什么我们可以使用多种方式调用axios

我们先不看具体请求逻辑是怎么实现的，先看看拦截器是怎么做到在请求前后预处理的。

## 拦截器（interceptor）

在上文`Axios`的构造器中我们可以看到

```js
this.interceptors = {
  request: new InterceptorManager(),
  response: new InterceptorManager()
};
```

`InterceptorManager`类用以管理拦截器

### InterceptorManager类

```js
function InterceptorManager() {
  this.handlers = []; // 存储拦截器函数
}

// 用以添加拦截器
InterceptorManager.prototype.use = function use(fulfilled, rejected, options) {
  this.handlers.push({
    fulfilled: fulfilled,
    rejected: rejected,
    synchronous: options ? options.synchronous : false, // 是否是同步执行
    runWhen: options ? options.runWhen : null // 指定拦截器在某种情况下执行
  });
  return this.handlers.length - 1;
};

// 删除拦截器
InterceptorManager.prototype.eject = function eject(id) {
  if (this.handlers[id]) {
    this.handlers[id] = null;
  }
};

// 将handler的每一项交由fn处理
InterceptorManager.prototype.forEach = function forEach(fn) {
  utils.forEach(this.handlers, function forEachHandler(h) {
    if (h !== null) {
      fn(h);
    }
  });
};
```

### request中InterceptorManager使用方式

```js
Axios.prototype.request = function request(config) {
    // 这里有合并 config 的操作，优先级是 defaults -> { method: "get" } -> this.defaults -> config
    // 也就是 库默认的配置 -> 默认为get请求 -> 创建Axios实例设置的配置 -> 请求的配置

    var requestInterceptorChain = []; // 请求拦截器链
    this.interceptors.request.forEach(function unshiftRequestInterceptors(interceptor) {
        // 无法通过runWhen条件的则不执行
        if (typeof interceptor.runWhen === 'function' && interceptor.runWhen(config) === false) {
            return;
        }

        // 以 fulfilled、rejected 成对unshift到 requestInterceptorChain
        requestInterceptorChain.unshift(interceptor.fulfilled, interceptor.rejected); 
    });

    var responseInterceptorChain = []; // 响应拦截器链
    this.interceptors.response.forEach(function pushResponseInterceptors(interceptor) {
        // 以 fulfilled、rejected 成对unshift到 responseInterceptorChain
        responseInterceptorChain.push(interceptor.fulfilled, interceptor.rejected);
    });

    var promise;

    // dispatchRequest是发送请求，第二个undefined是为了符合提取方式的占位值
    var chain = [dispatchRequest, undefined];

    // 将 请求拦截器 放置在dispatchRequest前面 
    Array.prototype.unshift.apply(chain, requestInterceptorChain);
    // 将 响应拦截器 放置在dispatchRequest后面
    chain.concat(responseInterceptorChain);

    // 通过此方式将config传入到请求拦截器的第一个fulfilled
    promise = Promise.resolve(config);
    while (chain.length) {
        // 因为在chain中fulfilled、rejected是成对存在的
        // 而 请求拦截器 必须返回config，下一个 请求拦截器 就会接受上一个config
        // 响应拦截器 会接受 dispatchRequest 的响应体或错误
        promise = promise.then(chain.shift(), chain.shift());
    }

    return promise;
}
```

大致流程就是如此

### synchronous

但是axios还为**请求拦截器**提供了`synchronous`选项，需要**全部**请求拦截器设置`synchronous`为true，那么就以同步的方式运行

```js
Axios.prototype.request = function request(config) {
  // ...
  
  var requestInterceptorChain = [];
  // 新增：用以标记是否是同步执行请求拦截器
  var synchronousRequestInterceptors = true;
  this.interceptors.request.forEach(function unshiftRequestInterceptors(interceptor) {
    if (typeof interceptor.runWhen === 'function' && interceptor.runWhen(config) === false) {
      return;
    }
	// 新增：需要 全部请求拦截器 设置synchronous成true才会同步执行
    synchronousRequestInterceptors = synchronousRequestInterceptors && interceptor.synchronous;

    requestInterceptorChain.unshift(interceptor.fulfilled, interceptor.rejected);
  });

  var responseInterceptorChain = [];
  this.interceptors.response.forEach(function pushResponseInterceptors(interceptor) {
    responseInterceptorChain.push(interceptor.fulfilled, interceptor.rejected);
  });

  var promise;
  // 新增：默认是异步的方式执行
  if (!synchronousRequestInterceptors) {
    var chain = [dispatchRequest, undefined];

    Array.prototype.unshift.apply(chain, requestInterceptorChain);
    chain.concat(responseInterceptorChain);

    promise = Promise.resolve(config);
    while (chain.length) {
      promise = promise.then(chain.shift(), chain.shift());
    }

    return promise;
  }

  // 新增：同步的方式执行请求拦截器
  var newConfig = config;
  while (requestInterceptorChain.length) {
    var onFulfilled = requestInterceptorChain.shift();
    var onRejected = requestInterceptorChain.shift();
    try {
      newConfig = onFulfilled(newConfig);
    } catch (error) {
      onRejected(error);
      break;
    }
  }

  try {
    promise = dispatchRequest(newConfig);
  } catch (error) {
    return Promise.reject(error);
  }

  while (responseInterceptorChain.length) {
    promise = promise.then(responseInterceptorChain.shift(), responseInterceptorChain.shift());
  }

  return promise;
};
```

### 启发：链式调用

```js
new Man("Hank").sleep(1).eat("supper").sleep(1).eat("me").sleepFirst(2);

// 打印结果
// 'This is Hank' -(等待2s)--> 'Wale up after 2' -(等待1s)--> 'Wale up after 1' 'Eat supper' -(等待1s)--> 'Eat me'
```

写法放在文末

## 发起请求dispatchRequest

在拿到请求拦截器处理后的config后

```js
// /lib/core/dispatchRequest.js
function dispatchRequest(config) {
  // 1.请求如果被取消则 throw cancel Error
  throwIfCancellationRequested(config);

  // 2.确保headers存在
  config.headers = config.headers || {};

  // 3.后面会讲：数据转化器转化请求data，也可以在config自定义配置transformRequest
  config.data = transformData.call(
    config,
    config.data,
    config.headers,
    config.transformRequest
  );

  // 4.扁平header设置，这里可以利用config.headers.get = {}设置某种请求类型的headers
  config.headers = utils.merge(
    config.headers.common || {},
    config.headers[config.method] || {},
    config.headers
  );

  // 5.删除无用header，因为在第四步时扁平化定义在headers了
  utils.forEach(
    ['delete', 'get', 'head', 'post', 'put', 'patch', 'common'],
    function cleanHeaderConfig(method) {
      delete config.headers[method];
    }
  );

  // 可以自定义适配器，默认为XHR（web）或http（node）适配器
  var adapter = config.adapter || defaults.adapter;

  return adapter(config).then(function onAdapterResolution(response) {
    // ...
    return response;
  }, function onAdapterRejection(reason) {
    // ...
    return Promise.reject(reason);
  });
};

function throwIfCancellationRequested(config) {
  if (config.cancelToken) {
    config.cancelToken.throwIfRequested();
  }
}
```

## 跨端实现（adapter）

### 适配器模式

- 是否有`XMLHttpRequest`对象来判断是否是web环境
- 是否有`process`对象来判断node环境

```js
// /lib/defaults.js
function getDefaultAdapter() {
  var adapter;
  if (typeof XMLHttpRequest !== 'undefined') {
    adapter = require('./adapters/xhr');
  } else if (typeof process !== 'undefined' && Object.prototype.toString.call(process) === '[object process]') {
    adapter = require('./adapters/http');
  }
  return adapter;
}
```

### 封装xhr

返回一个promise

```js
// /lib/adapters/xhr.js
function xhrAdapter(config) {
  return new Promise(function dispatchXhrRequest(resolve, reject) {
    // ...
    var request = new XMLHttpRequest();
    var fullPath = buildFullPath(config.baseURL, config.url);
    // buildURL: 格式化url; config.paramsSerializer: 序列化方式；
    request.open(config.method.toUpperCase(), buildURL(fullPath, config.params, config.paramsSerializer), true);
    request.timeout = config.timeout;
      
    request.onloadend = function onloadend() {
      // ...
      settle(resolve, reject, response); // 下面有代码
      request = null;
    };
    request.onabort = function handleError() {
      reject(/**/);
      request = null;
    };
    request.onerror = function handleError() {
      reject(/**/);
      request = null;
    };
	request.ontimeout = function handleTimeout() {
      reject(/**/);
	  request = null;
	};
    request.send(requestData);
  });
};
```

验证服务端的返回结果是否通过验证：

```js
// /lib/core/settle.js
function settle(resolve, reject, response) {
  var validateStatus = response.config.validateStatus; // 可以自定义成功规则
  if (!response.status || !validateStatus || validateStatus(response.status)) {
    resolve(response);
  } else {
    reject(/**/);
  }
};
```

同理将http封装成promise

## 取消请求（cancelToken）

### 如何使用

```js
import axios from 'axios'

// 第一种取消方法
axios.get(url, {
  cancelToken: new axios.CancelToken(cancel => {
    if (/* 取消条件 */) {
      cancel('取消日志');
    }
  })
});

// 第二种取消方法
const CancelToken = axios.CancelToken;
const source = CancelToken.source();
axios.get(url, {
  cancelToken: source.token
});
source.cancel('取消日志');

```

### 源码

```js
function CancelToken(executor) {
  if (typeof executor !== 'function') {
    throw new TypeError('executor must be a function.');
  }

  var resolvePromise;
  this.promise = new Promise(function promiseExecutor(resolve) {
    resolvePromise = resolve;
  });

  var token = this;
  executor(function cancel(message) { // 等到executor调用cancel方法，上面的this.promise就会执行下面的then回调
    if (token.reason) {
      return;
    }

    token.reason = new Cancel(message);
    resolvePromise(token.reason); 
  });
}

// 第二种方法的实现
CancelToken.source = function source() {
  var cancel;
  var token = new CancelToken(function executor(c) {
    cancel = c;
  });
  return {
    token: token,
    cancel: cancel
  };
};

// /lib/adapters/xhr.js 在send之前执行
if (config.cancelToken) {
  config.cancelToken.promise.then(function onCanceled(cancel) {
    if (!request) {
      return;
    }
    request.abort(); // xhr的取消请求
    reject(cancel);
    request = null;
  });
}
```

## 启发

### 链式调用

```js
class Man {
  constructor (name) {
    this.name = name;
    this.arr = [];
    console.log(`This is ${name}`);
    
    const executor = i => {
      if (i < this.arr.length) {
        this.arr[i]().then(value => {
          console.log(value);
          executor(i + 1);
        });
      }
    }
    
    Promise.resolve().then(()=> {
      executor(0)
    }, 0);
  }
  
  sleep(sec) {
    this.arr.push(() => {
      return new Promise(resolve => {
        setTimeout(() => {
          resolve(`Wale up after ${sec}`);
        }, sec * 1000);
      })
    })
    return this;
  }
  
  eat(food) {
    this.arr.push(() => {
      return Promise.resolve(`Eat ${food}`);
    })
    return this;
  }
  
  sleepFirst(sec) {
    this.arr.unshift(() => {
      return new Promise(resolve => {
        setTimeout(() => {
          resolve(`Wale up after ${sec}`);
        }, sec * 1000);
      })
    })
    
    return this;
  }
}
```

## 参考

[Axios源码深度剖析](https://github.com/ronffy/axios-tutoria)

