---
title: React Router
date: 2021-05-28 20:13:21
tags:
  - React
categories:
  - 前端
---
- react-router：路由核心库，包含诸多和路由功能相关的核心代码
- react-router-dom：利用路由核心库，结合实际的页面，实现跟页面路由密切相关的功能

## 安装

```bash
# react-router-dom依赖react-router
# 安装的时候会把react-router一起安装了
npm install react-router-dom -S
```
<!--more-->
## 1.两种路由模式 Hash、History

- `HashRouter`：使用 hash 模式匹配
- `BrowserRouter`：使用 BrowserHistory 模式匹配

两者区别
1.底层原理不一样：
	BrowserRouter使用的是H5的history API，不兼容IE9及以下版本。
	HashRouter使用的是URL的哈希值。
2.path表现形式不一样
	BrowserRouter的路径中没有#，例如：localhost:3000/demo/test
	HashRouter的路径包含#,例如：localhost:3000/#/demo/test
3.刷新后对路由state参数的影响
	BrowserRouter没有任何影响，因为state保存在history对象中。
	HashRouter刷新后会导致路由state参数的丢失！！！
4.备注：HashRouter可以用于解决一些路径错误相关的问题。

## 2. 跳转组件 Link和NavLink

Link组件

```jsx
<Link to="/xxxxx">Demo</Link>
```

Link组件配置方式

- to：要跳转的链接（String | Object  | Function）

- replace：设置为true，则跳转替换路由栈顶

- innerRef：可以将内部的a元素的ref附着在传递的对象或函数参数上（Function | RefObject）

- component：可以设置自定义的导航组件

	```jsx
	const FancyLink = React.forwardRef((props, ref) => (
	    <a ref={ref} {...props}>💅 {props.children}</a>
	))
	
	<Link to="/" component={FancyLink} />
	```

**NavLink**可以实现路由链接的高亮，通过`activeClassName`指定样式类名

```jsx
<NavLink to="/xxxxx">Demo</NavLink>
```

## 3. route组件

配置方式

- path：匹配的路径
	- 默认情况下，不区分大小写，可以设置sensitive属性为true，来区分大小写
	- 默认情况下，只匹配初始目录，如果要精确匹配，配置exact属性为true
	- 如果不写path，则会匹配任意路径
- component：匹配成功后要显示的组件
- children
	- 传递React元素，无论是否匹配，一定会显示children，并且会忽略component属性
	- 传递一个函数，该函数有多个参数，这些参数来自于上下文，该函数返回react元素，则一定会显示返回的元素，并且忽略component属性
- render：匹配成功渲染的组件，内联书写render，而不是创建一个React元素
- sensitive：true则对paht的大小写敏感
- exact：精确匹配

### route嵌套

下述代码中，用户访问`/repos`时，会先加载`App`组件，然后在它的内部再加载`Repos`组件。

```jsx
<BrowserRouter history={hashHistory}>
    <Route path="/" component={App}>
        <Route path="/repos" component={Repos}/>
        <Route path="/about" component={About}/>
    </Route>
</BrowserRouter>
```

### 匹配规则

1. :paramName

`:paramName`匹配URL的一个部分，直到遇到下一个`/`、`?`、`#`为止。这个路径参数可以通过`this.props.params.paramName`取出。

2. ()

`()`表示URL的这个部分是可选的。

3. \*

`*`匹配任意字符，直到模式里面的下一个字符为止。匹配方式是非贪婪模式。

4. \**

`**` 匹配任意字符，直到下一个`/`、`?`、`#`为止。匹配方式是贪婪模式。

此外，URL的查询字符串`/foo?bar=baz`，可以用`this.props.location.query.bar`获取。

```jsx
import { BrowserRouter, Route } from "react-router-dom";

<BrowserRouter>
    <Route path="/a" component={A} />
    <Route path="/a/d" exact component={D}/>  // 精确匹配
    <Route component={C} />
    <Route path='/abc' children={E}/>
</BrowserRouter>
```

  上述代码，如果路由是 **/a/d，则会渲染 A、D、C、E**，react router是会从上至下完全匹配一遍，而为了解决这个问题，引入了Switch组件。

## 4. Switch组件

会**循环所有子元素**，当匹配到第一个Route后，会立即停止匹配并渲染，因此不能在Switch组件的子元素中使用除Route外的其他组件

```jsx
<Router>
    <Switch>
        <Route path="/a" component={A} />
        <Route path="/a/b" component={B} />
        <Route component={C} />
    </Switch>
</Router>
```

## 5. Redirect组件

一般写在所有路由注册的最下方，当所有路由都无法匹配时，跳转到Redirect指定的路由

```jsx
<Redirect to="/about/" />
```

## 6. 路由信息

Router组件会创建一个上下文，并且向上下文中注入一些信息，该上下文对开发者是隐藏的，Route组件若匹配到了地址，则会将这些上下文中的信息作为属性传入对应的组件

```jsx
import React from 'react'
import { BrowserRouter as Router, Route, Switch } from "react-router-dom"

// a
function A(props) {
    console.log(props)
    return <h1>组件A</h1>
}


export default function App() {
    return (
        <Router>
            <Switch>
                <Route path="/a" component={A} />
            </Switch>
        </Router>
    )
}
```

打印一下props

```bash
history:
	go: ƒ go(n)
	goBack: ƒ goBack()
	goForward: ƒ goForward()
	push: ƒ push(path, state)
	replace: ƒ replace(path, state)
location:
    hash：页面hash
    pathname：页面的path
    state：push时传入的数据
    search：传入的参数
match:
	isExact：事实上，当前的路径和路由配置的路径是否是精确匹配的
	params：获取路径中对应的数据
	path：路径规则
	url：页面路径
```

## 7.路由传参

### 5.1 params 参数

路由链接(携带参数)：`<Link to='/demo/test/tom/18'}>详情</Link>`
注册路由(声明接收)：`<Route path="/demo/test/:name/:age" component={Test}/>`
接收参数：this.props.match.params

### 5.2 search参数

路由链接(携带参数)：`<Link to='/demo/test?name=tom&age=18'}>详情</Link>`
注册路由(无需声明，正常注册即可)：`<Route path="/demo/test" component={Test}/>`
接收参数：`this.props.location.search`
备注：获取到的search是urlencoded编码字符串，需要借助querystring解析

### 5.3 state参数

路由链接(携带参数)：`<Link to={{pathname:'/demo/test',state:{name:'tom',age:18}}}>详情</Link>`
注册路由(无需声明，正常注册即可)：`<Route path="/demo/test" component={Test}/>`
接收参数：`this.props.location.state`
备注：刷新也可以保留住参数

## 8. 编程式导航

借助`this.props.history`上的方法，基本和H5的history一致，就只是列一下了

```jsx
this.props.history.push()
this.props.history.replace()
this.props.history.goBack()
this.props.history.goForward()
this.props.history.go()
```

## 9. withRouter

> 由于**只能在跳转的组件中获取路由信息**，所以提出`withRouter`，用withRouter包裹的组件能获取路由信息

```jsx
function ButtonContainer(props) {
    let history = props.history;
    return (
        <div className={"container"}>
            <button onClick={() => {history.push("/a?key=a", "A状态数据")}}>A</button>
            <button onClick={() => {history.push("/a/b?key=b", "B状态数据")}}>B</button>
            <button onClick={() => {history.push("/a/c?key=c", "C状态数据")}}>C</button>
            <button onClick={() => {history.push("/a/d?key=d", "D状态数据")}}>D</button>
            <button onClick={() => {history.push("/a/news/2020/12/21")}}>News</button>
        </div>
    )
}

const WithRouterButtonContainer = withRouter(ButtonContainer)
```

其返回值是一个新组件

## 参考

[react学习笔记5-React Router](https://blog.sakura-snow.com/post/react-node-5/)
[React Router 使用教程](
