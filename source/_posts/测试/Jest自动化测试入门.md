---
title: Jest自动化测试入门
date: 2020-11-17 15:55:41
tags:
  - js
categories:
  - 测试
---
## 1. 环境搭建

```
npm install --save-dev jest
```

然后在项目根目录下，控制台执行如下命令，就会初始化jest配置 jest.config.js

```
npx jest --init
```
<!--more-->
生成代码覆盖率

```
npx jest --coverage
```

![image-20201117155410176](image-20201117155410176.png)

## 2.简单测试

首先有两个文件

liang.js

```js
function foo(money) {
  return money > 10000 ? '有钱哩' : '难受';
}

function bar(money) {
  return money > 1000 ? '吃大餐' : '吃空气';
}

module.exports = {
  foo,
  bar
}
```

liang.test.js

```js
const { foo, bar } = require('./liang');

test('foo方法-5000', () => {
  // toBe ~= ===
  expect(foo(5000)).toBe('难受')
})

test('bar方法-2000', () => {
  expect(bar(2000)).toBe('吃大餐')
})
```

把待测试函数包在`expect`中，用匹配器（这里是`toBe`）进行检查输出

然后在`package.json`中添加运行指令

```json
{
  "scripts": {
    "test": "jest"	// 想要有监视功能就加上 --watchAll
  }
}
```

然后`npm run test`就能运行我们第一个jest测试了

![image-20201116165402179](image-20201116165402179.png)

## 3.常用匹配器

| 匹配器        | 功能                                 |
| ------------- | ------------------------------------ |
| toBe          | ===                                  |
| toEqual       | 递归比较对象所有属性的值（深度相等） |
| toBeNull      | 只匹配 `null`                        |
| toBeUndefined | 只匹配 `undefined`                   |
| toBeDefined   | 不为`undefined`                      |
| toBeTruthy    | 匹配任何 `if` 语句为真               |
| toBeFalsy     | 匹配任何 `if` 语句为假               |

可以在上述匹配器前加个not表示取反，比如`.not.toBe`表示不严格等于

有关匹配器的完整列表，请查阅 [参考文档](https://jestjs.io/docs/zh-Hans/expect)

## 4. 测试异步方法

### 4.1 引入文件

fetchData.js

```js
import axios from 'axios'

export function fetchData () {
  return axios.get('http://localhost:3000/');
}
```

fetchData.test.js

```js
import { fetchData } from './fetchData'

test('fetchData 方法测试1', () => {
  fetchData().then(response => {
    expect(response.data).toEqual({
      success: true,
    })
  })
})
```

**无论怎么样都会测试成功，因为jest测试一旦执行到末尾就会完成**
所以问题再与一旦`fetchData`执行结束，此测试就在没用调用回调函数前结束

### 4.2 解决方法

```js
test('fetchData 方法测试2', (done) => {
  fetchData().then(response => {
    expect(response.data).toEqual({
      success: true,
    })
    done(); // Jest会等 done 回调函数执行结束后，结束测试，所以会
  })
})
```

### 4.3 export数量断言

下面一段代码的功能：检查接口是否存在，如果不存在就测试成功

```js
test('fetchData catch方法测试', () => {
  expect.assertions(1) // 断言，必须执行一次export，如果不执行则报错
  // 如果不加断言，那么如果没有错误被catch到，则测试不会被执行，显示不出报错
  fetchData().catch(err => {
    expect(err.toString().indexOf('404') > -1).toBe(true);
  })
})
```

### 4.4 async和await的测试方法

如果是用await进行测试的话，可以使用`export`的`resolves`和`rejects`

```js
test('fetchData async方法', async () => {
  await expect(fetchData()).resolves.toMatchObject({
    data: {
      success: true,
    }
  })
})
```

## 5. 钩子函数

### 5.1 功能描述

像vue-router的导航守卫一样的，jest也有四个钩子函数

| 钩子函数   | 功能                             |
| ---------- | -------------------------------- |
| beforeAll  | 在**所有**测试用例**之前**执行   |
| afterAll   | 在所有测试用例**之后**执行       |
| beforeEach | 在**每一个**测试用例之前执行     |
| afterEach  | 在**每一个**测试用例**之后**执行 |

### 5.2 作用域

默认情况下`before` 和 `after` 的块可以应用到文件中的每个测试，可以使用`describe`声明一个作用域，这些钩子函数在`describe`声明 的作用域内调用，则只会作用在该作用域内。

```js
import { foo, bar } from '../simpleDemo/liang'

beforeAll(() => {
  console.log("我是在外面的beforeAll")
})

beforeEach(() => {
  console.log("我是在外面的beforeEach")
})

describe('describe inner', () => {
  beforeAll(() => {
    console.log("我是在里面的beforeAll")
  })

  test('foo方法-5000', () => {
    // toBe ~= ===
    expect(foo(5000)).toBe('难受')
  })

  test('bar方法-2000', () => {
    expect(bar(2000)).toBe('吃大餐')
  })
})
```

![image-20201117153234402](image-20201117153234402.png)

### 5.3 钩子函数的作用规则

由上面截图的执行顺序，可以引出以下三条规则

- 钩子函数在父级分组可作用于子集
- 钩子函数同级分组作用域互不干扰，各起作用
- 先执行外部的钩子函数，再执行内部的钩子函数