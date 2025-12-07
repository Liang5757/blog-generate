---
title: npm script
date: 2023-10-06 19:27:17
tags:
  - js
categories:
  - 前端
---

## 串行 or 并行

并行跑命令：&，在命令结尾加`& wait`，可以使用ctrl c关闭命令行来结束进程
串行跑命令：&&

## 控制日志输出

- --silent（-s）：输出尽可能少的日志
- --verbose：显示尽可能多的状态，日志级别的输出，用于调试

<!--more-->

## npm自动补全

> 不支持workspace的仓库

npm官方支持补全：[npm-completion](https://docs.npmjs.com/cli/v9/commands/npm-completion)

第 1 步，把 npm completion 产生的命令放在单独的文件中：

```bash
npm completion >> ~/.npm-completion.bash
```

第 2 步，在 .bashrc 或者 .zshrc 中引入这个文件：

```bash
echo "[ -f ~/.npm-completion.bash ] && source ~/.npm-completion.bash;" >> ~/.bashrc
echo "[ -f ~/.npm-completion.bash ] && source ~/.npm-completion.bash;" >> ~/.zshrc
```

最后执行 `source ~/.zshrc` 或者 `source ~/.bashrc`，来让自动完成生效。

使用补全方式：尝试在命令行中输入 npm run，**然后键入空格（空格很重要）**，然后键入 tab 键

## 一些跨平台的第三方库

- [rimraf](https://github.com/isaacs/rimraf) 或 [del-cli](https://www.npmjs.com/package/del-cli)，用来删除文件和目录，实现类似于 `rm -rf` 的功能；
- [cpr](https://www.npmjs.com/package/cpr)，用于拷贝、复制文件和目录，实现类似于 `cp -r` 的功能；
- [make-dir-cli](https://www.npmjs.com/package/make-dir-cli)，用于创建目录，实现类似于 `mkdir -p` 的功能；

### 跨平台的引用变量

Linux 下直接可以用 `$npm_package_name`，而 Windows 下必须使用 `%npm_package_name%`，我们可以使用 [cross-var](https://www.npmjs.com/package/cross-var) 

```bash
npm i cross-var -D
```

#### 使用方式

```bash
  "scripts": {
-    "cover:open": "opn http://localhost:$npm_package_config_port",
+    "cover:open": "cross-var opn http://localhost:$npm_package_config_port",
   },
```

引入 cross-var 之后，它竟然给我们安装了 babel，如果想保持依赖更轻量的话，可以考虑使用 [cross-var-no-babel](https://link.juejin.cn/?target=https%3A%2F%2Fwww.npmjs.com%2Fpackage%2Fcross-var-no-babel)

### 跨平台的环境变量

因为不同平台的环境变量语法不同，我们可以使用 [cross-env](https://www.npmjs.com/package/cross-env) 来实现 npm script 的跨平台兼容

```bash
npm i cross-env -D
```

#### 使用方式

```bash
  "scripts": {
-    "test": "NODE_ENV=test mocha tests/",
+    "test": "cross-env NODE_ENV=test mocha tests/",
  },
```

## 抽离script到单独文件

当 npm script 不断累积、膨胀的时候，全部放在 package.json 里面可能并不是个好主意，因为这样会导致 package.json 糟乱，可读性降低。

借助 [scripty](https://github.com/testdouble/scripty) 我们可以将 npm script 剥离到单独的文件中，从而把复杂性隔到单独的模块里面，让代码整体看起来更加清晰。

```bash
npm i scripty -D
```

### 使用方式

执行`npm run foo:bar`会向`scripts/foo/bar`寻找可执行脚本并执行

```json
"scripts": {
  "foo:bar": "scripty"
}
```

## 钩子

npm script 的设计者为命令的执行增加了类似生命周期的机制，具体来说就是 `pre` 和 `post` 钩子脚本

举例来说，运行 npm run test 的时候，分 3 个阶段：

1. 检查 scripts 对象中是否存在 pretest 命令，如果有，先执行该命令；
2. 检查是否有 test 命令，有的话运行 test 命令，没有的话报错；
3. 检查是否存在 posttest 命令，如果有，执行 posttest 命令；

## 在 Git Hooks 中执行 npm script

```bash
npm install -D husky
```

在package.json增加

```json
{
  "scripts": {
    "precommit": "eslint src/**/*.js"
  }
}
```

但是这样会检查到项目中所有的lint错误，如何只检查提交的文件呢？

需要用到[lint-staged](https://link.juejin.cn/?target=https%3A%2F%2Fgithub.com%2Fokonet%2Flint-staged)

```bash
npm install -D lint-staged
```

package.json改为如下，就可以只lint提交的文件啦

```json
{
  "scripts": {
    "precommit": "lint-staged"
  },
  "lint-staged": {
    "src/**/*.js": "eslint"
  }
}
```

## 参考文章

[用 husky 和 lint-staged 构建超溜的代码检查工作流](https://juejin.cn/post/6844903479283040269)