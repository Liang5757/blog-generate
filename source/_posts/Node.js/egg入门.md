---
title: egg入门
date: 2021-07-18 23:28:05
tags:
  - egg.js
categories:
  - 后端
---
## 快速初始化

用npm

```bash
npm init egg --type=simple
```
<!--more-->
## 目录结构

```
egg-project
├── package.json
├── app.js (可选) 			# 用于自定义启动时的初始化工作
├── agent.js (可选) 			# 用于自定义启动时的初始化工作
├── app
|   ├── router.js 			 # 用于配置 URL 路由规则
│   ├── controller 			 # 用于解析用户的输入，处理后返回相应的结果
│   |   └── home.js
│   ├── service (可选) 		# 用于编写业务逻辑层
│   |   └── user.js
│   ├── middleware (可选) 	# 用于编写中间件
│   |   └── response_time.js
│   ├── schedule (可选)		# 用于定时任务
│   |   └── my_task.js
│   ├── public (可选) 		# 用于放置静态资源
│   |   └── reset.css
│   ├── view (可选)			# 用于放置模板文件
│   |   └── home.tpl
│   └── extend (可选) 		# 用于框架的扩展
│       ├── helper.js (可选)
│       ├── request.js (可选)
│       ├── response.js (可选)
│       ├── context.js (可选)
│       ├── application.js (可选)
│       └── agent.js (可选)
├── config 					# 用于编写配置文件
|   ├── plugin.js 			 # 用于配置需要加载的插件
|   ├── config.default.js
│   ├── config.prod.js
|   ├── config.test.js (可选)
|   ├── config.local.js (可选)
|   └── config.unittest.js (可选)
└── test 					# 用于单元测试
    ├── middleware
    |   └── response_time.test.js
    └── controller
        └── home.test.js
```

## 路由（Router）

> 将用户的请求基于 method 和 URL 分发到了对应的 Controller 上，感觉api和Koa差不多。

### 参数获取

#### query

就列一个query参数的获取，需要注意的是，app.controller.search与文件相对应（也就是app/controller/search.js）

```js
// curl http://127.0.0.1:7001/search?name=egg

// app/router.js
module.exports = app => {
  app.router.get('/search', app.controller.search.index);
};

// app/controller/search.js
exports.index = async ctx => {
  ctx.body = `search: ${ctx.query.name}`; 
};
```

**为了避免用户恶意传递重复key的query参数导致系统报错，ctx.query只取第一次出现的值，并且保证一定是字符串类型**

如果必须要重复的key，比如需要传递数组：id=1&id=2&id=3，可以用`ctx.queries`，其也保证了一定是数组类型

#### body

框架内置了 [bodyParser](https://github.com/koajs/bodyparser) 中间件来对JSON和Form-Data的请求 body 解析成 object 挂载到 `ctx.request.body` 上。

body 最大长度为 `100kb`，超出长度413状态码，可以在 `config/config.default.js` 中覆盖框架的默认值

```js
module.exports = {
  bodyParser: {
    jsonLimit: '1mb',
    formLimit: '1mb',
  },
};
```

ctx.body是ctx.response.body的简写

#### header

可以通过`ctx.get(name)`获取，其会自动处理大小写

#### Cookie

获取、设置方式和Koa一样，配置方式如下

```js
module.exports = {
  cookies: {
    // httpOnly: true | false,
    // sameSite: 'none|lax|strict',
  },
};
```

#### Session

框架内置 [Session](https://github.com/eggjs/egg-session) 插件，可以通过`ctx.session`，具体配置可以看 [Cookie 与 Session](https://eggjs.org/zh-cn/core/cookie-and-session.html#session)

### 参数校验

借助 [Validate](https://github.com/eggjs/egg-validate) 插件提供便捷的参数校验机制，配置方式如下

```js
// config/plugin.js
exports.validate = {
  enable: true,
  package: 'egg-validate',
};
```

ctx上有一个validate方法，可以传入校验配置来对参数进行校验

```js
// curl -X POST http://127.0.0.1:7001/user --data 'username=abc@abc.com&password=111111&re-password=111111'

// app/router.js
module.exports = app => {
  app.router.post('/user', app.controller.user);
};

// app/controller/user.js
const createRule = {
  username: {
    type: 'email',
  },
  password: {
    type: 'password',
    compare: 're-password', // 与re-password比较，是否一致
  },
};

exports.create = async ctx => {
  // 如果校验报错，会抛出异常
  ctx.validate(createRule);
  ctx.body = ctx.request.body;
};
```

### 重定向

#### 内部重定向

```js
app.router.redirect('/', '/home/index', 302);
```

####  外部重定向

```js
ctx.redirect(`http://cn.bing.com`);
```

### resources生成CRUD路由结构

```js
// app/router.js
module.exports = app => {
    const { router, controller } = app;
    router.resources('posts', '/api/posts', controller.posts);
};
```

接下来在对应文件下实现对应函数就可以了

```js
// app/controller/posts.js
exports.index = async () => {};

exports.new = async () => {};

exports.create = async () => {};

exports.show = async () => {};

exports.edit = async () => {};

exports.update = async () => {};

exports.destroy = async () => {};
```

| Method | Path            | Route Name | Controller.Action             |
| ------ | --------------- | ---------- | ----------------------------- |
| GET    | /posts          | posts      | app.controllers.posts.index   |
| GET    | /posts/new      | new_post   | app.controllers.posts.new     |
| GET    | /posts/:id      | post       | app.controllers.posts.show    |
| GET    | /posts/:id/edit | edit_post  | app.controllers.posts.edit    |
| POST   | /posts          | posts      | app.controllers.posts.create  |
| PUT    | /posts/:id      | post       | app.controllers.posts.update  |
| DELETE | /posts/:id      | post       | app.controllers.posts.destroy |

### 模块化

```js
// app/router.js
module.exports = app => {
  require('./router/news')(app);
  require('./router/admin')(app);
};

// app/router/news.js
module.exports = app => {
  app.router.get('/news/list', app.controller.news.list);
};

// app/router/admin.js
module.exports = app => {
  app.router.get('/admin/user', app.controller.admin.user);
};
```

## 控制器（Controller）

> 解析用户的输入，处理后返回相应的结果，框架建议在 controller 对请求参数进行处理（校验、转换），然后调用对应的 service 方法处理业务

```js
// app/controller/post.js
const Controller = require('egg').Controller;

class PostController extends Controller {
  async create() {
    const { ctx, service } = this;
    const createRule = {
      title: { type: 'string' },
      content: { type: 'string' },
    };
      
    // 校验参数
    ctx.validate(createRule);
    // 组装参数
    const author = ctx.session.userId;
    const req = Object.assign(ctx.request.body, { author });
    // 调用 Service 进行业务处理
    const res = await service.post.create(req);
    // 设置响应内容和响应状态码
    ctx.body = { id: res.id };
    ctx.status = 201;
  }
}

module.exports = PostController;
```

### 获取上传的文件

#### file模式

在config中配置file模式

```json
// config/config.default.js
exports.multipart = {
  mode: 'file',
};
```

controller层代码

```js
// app/controller/upload.js
const Controller = require('egg').Controller;
const fs = require('mz/fs');

module.exports = class extends Controller {
  async upload() {
    const { ctx } = this;
    console.log(ctx.request.body);
    console.log('got %d files', ctx.request.files.length);
    for (const file of ctx.request.files) { // files获取上传的多文件
      console.log('field: ' + file.fieldname);
      console.log('filename: ' + file.filename);
      console.log('encoding: ' + file.encoding);
      console.log('mime: ' + file.mime);
      console.log('tmp filepath: ' + file.filepath);
      let result;
      try {
        // 处理文件，比如上传到云端
        result = await ctx.oss.put('egg-multipart-test/' + file.filename, file.filepath);
      } finally {
        // 需要删除临时文件
        await fs.unlink(file.filepath);
      }
      console.log(result);
    }
  }
};
```

#### stream模式

```js
const path = require('path');
const sendToWormhole = require('stream-wormhole');
const Controller = require('egg').Controller;

class UploaderController extends Controller {
  async upload() {
    const ctx = this.ctx;
    const stream = await ctx.getFileStream();
    const name = 'egg-multipart-test/' + path.basename(stream.filename);
    // 文件处理，上传到云存储等等
    let result;
    try {
      result = await ctx.oss.put(name, stream);
    } catch (err) {
      // 必须将上传的文件流消费掉，要不然浏览器响应会卡死
      await sendToWormhole(stream);
      throw err;
    }

    ctx.body = {
      url: result.url,
      // 所有表单字段都能通过 `stream.fields` 获取到
      fields: stream.fields,
    };
  }
}

module.exports = UploaderController;
```

## Service层

其不是单例，在每次请求时访问`ctx.service.xx` 时延迟实例化，所以可以用`this.ctx`获取每一次请求的上下文

```js
// app/service/user.js
const Service = require('egg').Service;

class UserService extends Service {
  async find(uid) {
    const user = await this.ctx.db.query('select * from user where uid = ?', uid);
    return user;
  }
}

module.exports = UserService;
```

函数return的值会返回给前端

## 定时任务

所有的定时任务都统一存放在 `app/schedule` 目录下

```js
const Subscription = require('egg').Subscription;

class UpdateCache extends Subscription {
  // 通过 schedule 属性来设置定时任务的执行间隔等配置
  static get schedule() {
    return {
      interval: '1m', // 1 分钟间隔
      type: 'all', // 指定所有的 worker 都需要执行
    };
  }

  // subscribe 是真正定时任务执行时被运行的函数
  async subscribe() {
    const res = await this.ctx.curl('http://www.api.com/cache', {
      dataType: 'json',
    });
    this.ctx.app.cache = res.data;
  }
}

module.exports = UpdateCache;
```

还可以简写为

```js
module.exports = {
  schedule: {
    interval: '1m', // 1 分钟间隔
    type: 'all', // 指定所有的 worker 都需要执行
  },
  async task(ctx) {
    const res = await ctx.curl('http://www.api.com/cache', {
      dataType: 'json',
    });
    ctx.app.cache = res.data;
  },
};
```

没有用过，具体配置看官方文档 [定时任务](https://eggjs.org/zh-cn/basics/schedule.html)

## egg-mysql

### 安装与配置

```bash
npm i --save egg-mysql
```

开启插件

```json
// config/plugin.js
exports.mysql = {
  enable: true,
  package: 'egg-mysql',
};
```

数据库配置

```json
// config/config.${env}.js
exports.mysql = {
  // 单数据库信息配置
  clients: {
    // clientId, 获取client实例，需要通过 app.mysql.get('clientId') 获取
    db1: {
      host: 'mysql.com',
      port: '3306',
      user: 'test_user',
      password: 'test_password',
      database: 'test',
    },
    db2: {
      host: 'mysql2.com',
      port: '3307',
      user: 'test_user',
      password: 'test_password',
      database: 'test',
    },
  }
  // 所有数据库配置的默认值
  default: {},
  // 是否加载到 app 上，默认开启
  app: true,
  // 是否加载到 agent 上，默认关闭
  agent: false,
};
```

### 使用方式：

```js
const client1 = app.mysql.get('db1');
await client1.query(sql, values);

const client2 = app.mysql.get('db2');
await client2.query(sql, values);
```

