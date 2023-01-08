---
title: redux入门
date: 2021-06-15 21:02:58
tags:
  - React
categories:
  - 前端
---
> Redux 官方文档对 Redux 的定义是：**一个可预测的 JavaScript 应用状态管理容器**

## 安装

```bash
npm install --save redux
```
<!--more-->
## Store

> Redux应用只有单一的 store

通过`createStore()`创建一个store

```jsx
import { createStore } from 'redux'
import todoApp from './reducers' // 这个在下面会讲
let store = createStore(todoApp)
```

`createStore()`的第二个参数是可选的, 用于设置 state 初始状态。这对开发同构应用时非常有用，服务器端 redux 应用的 state 结构可以与客户端保持一致, 那么客户端可以将从网络接收到的服务端 state 直接用于本地数据初始化。

提供四个函数

- `getState()`方法获取 state
- `dispatch(action)`方法分发 action 更新 state
- `subscribe(listener)`注册监听器，返回的函数注销监听器
- `replaceReducer(nextReducer)` 一般在 Webpack Code-Splitting 按需加载的时候用

redux规定禁止直接修改 state，也就是下面的写法

```jsx
var state = store.getState()
state.counter = state.counter + 1 // 禁止在业务逻辑中直接修改 state
```

只能通过 dispatch 一个 action 来修改

## Action

是把数据从应用传到 store 的有效载荷。它是 store 数据的**唯一**来源。一般来说你会通过 [`store.dispatch()`](https://cn.redux.js.org/docs/api/Store.html#dispatch) 将 action 传到 store。

### 1 Action格式

除了`type`字段外，action的结构完全由你自己决定，一般参照 [Flux 标准 Action](https://github.com/acdlite/flux-standard-action) 获取关于如何构造 action 的建议。

```js
const ADD_TODO = 'ADD_TODO'
// 下面是常见的action格式
{
  type: ADD_TODO,
  text: 'Build my first Redux app'
}
```

### 2 Action创建函数

就是生成action的方法

```js
function addTodo(text) {
    return {
        type: ADD_TODO,
        text
    }
}
```

### 3 dispatch

只需把 action 创建函数的结果传给 `dispatch()` 方法即可发起一次 dispatch 过程

```js
dispatch(addTodo(text))
```

## Reducer

指定了应用状态的变化如何**响应** actions 并发送状态到 store 的，记住 actions 只是描述了*有事情发生了*这一事实，并没有描述应用如何更新 state。

reducer就是一个纯函数，**接收旧的 state 和 action，返回新的 state**。

```js
;(previousState, action) => newState
```

```jsx
var initState = {
    counter: 0,
    todos: []
}

function reducer(state, action) {
    // 应用的初始状态是在第一次执行 reducer 时设置的 ※
    if (!state) state = initState

    switch (action.type) {
        case 'ADD_TODO':
            var nextState = _.cloneDeep(state) // 用到了 lodash 的深克隆
            nextState.todos.push(action.payload) 
            return nextState
        case 'ADD_COUNT':
            return Object.assign({}, state, {
                visibilityFilter: action.filter
            })
        default:
            // 由于 nextState 会把原 state 整个替换掉
            // 若无修改，必须返回原 state（否则就是 undefined）
            return state
    }
}
```

注意：

- **不要修改 `state`。** 使用 [`Object.assign()`](https://developer.mozilla.org/en/docs/Web/JavaScript/Reference/Global_Objects/Object/assign) 新建了一个副本。不能这样使用 `Object.assign(state, { visibilityFilter: action.filter })`，因为它会改变第一个参数的值
- **在 `default` 情况下返回旧的 `state`。**否则 state 会变成`undefined`

## 拆分Reducer

当业务逻辑复杂时，所有状态聚合在一个reducer函数里处理，逻辑会变得相当复杂。

我们可以提出一个主reducer函数，它调用多个子 reducer 分别处理子 state 中的数据，然后再把这些数据合成一个大的单一对象。

主 reducer 并不需要设置初始化时完整的 state。初始时，如果传入 `undefined`, **子 reducer 将负责返回它们的默认值**

```js
function todos(state = [], action) {
  switch (action.type) {
    case ADD_TODO:
      return [
        ...state,
        {
          text: action.text,
          completed: false
        }
      ]
    case TOGGLE_TODO:
      return state.map((todo, index) => {
        if (index === action.index) {
          return Object.assign({}, todo, {
            completed: !todo.completed
          })
        }
        return todo
      })
    default:
      return state
  }
}

function visibilityFilter(state = SHOW_ALL, action) {
  switch (action.type) {
    case SET_VISIBILITY_FILTER:
      return action.filter
    default:
      return state
  }
}

// 主reducer
function todoApp(state = {}, action) {
  return {
    visibilityFilter: visibilityFilter(state.visibilityFilter, action),
    todos: todos(state.todos, action)
  }
}
```

可以利用`combineReducers()`来简化代码

```js
import { combineReducers } from 'redux'

const todoApp = combineReducers({
  visibilityFilter,
  a: todos
})

export default todoApp
```

注意上面的写法和下面完全等价：

```js
export default function todoApp(state = {}, action) {
  return {
    visibilityFilter: visibilityFilter(state.visibilityFilter, action),
    todos: todos(state.todos, action)
  }
}
```

## 异步数据流

如果只是简单的redux store是不支持用dispatch异步更新store，可以使用`react-thunk`来增强

### 安装

```bash
npm install redux-thunk
```

### 使用applyMiddleware引入中间件

> 与Koa和Express类似，redux也提供了注册中间件的方法：`applyMiddleware`，这个中间件执行时间是在**dispatch一个action之后，到达reducer之前**，执行顺序是从上到下，传递action

```jsx
import thunk from 'redux-thunk'
import { createStore, applyMiddleware } from 'redux'
import rootReducer from './reducers'

const store = createStore(
  rootReducer,
  applyMiddleware(
    thunk, // 允许我们 dispatch() 函数
  )
)
```

### 动机

引入后**允许我们`dispatch`一个函数**，这个函数内部可以`dispatch`，这个函数接受两个参数，第一个是`dispatch`，第二个是`getState`，函数内部允许放一些异步操作，来解决redux只能同步`dispatch(action)`的问题

### react-thunk 源码

出乎意料的简单

```jsx
function createThunkMiddleware(extraArgument) {
    return ({ dispatch, getState }) => (next) => (action) => {
        if (typeof action === 'function') { // 如果是函数，当作函数执行
            return action(dispatch, getState, extraArgument);
        }

        return next(action);
    };
}

const thunk = createThunkMiddleware();
thunk.withExtraArgument = createThunkMiddleware;

export default thunk;
```

当然我们也可以用`redux-promise`这个中间件来`dispatch`一个`promise`

### redux-promise 源码

```js
import isPromise from 'is-promise';
import { isFSA } from 'flux-standard-action';

export default function promiseMiddleware({ dispatch }) {
    return next => action => {
        // 判断是不是 flux 标准的action
        if (!isFSA(action)) {
            return isPromise(action) ? action.then(dispatch) : next(action);
        }

        return isPromise(action.payload)
            ? action.payload
            .then(result => dispatch({ ...action, payload: result }))
            .catch(error => {
            dispatch({ ...action, payload: error, error: true });
            return Promise.reject(error);
        })
        : next(action);
    };
}
```

## 与react配合使用

### 安装

```bash
npm install --save react-redux
```

### 使用方式

利用 Provider 组件分发store状态

```jsx
// ...
import store from './app/store'
import { Provider } from 'react-redux'

ReactDOM.render(
  <Provider store={store}>
    <App />
  </Provider>,
  document.getElementById('root')
)
```

在组件中使用`useSelector`，`useDispatch`来获取 state 和分发 action

```jsx
// ...
import { useSelector, useDispatch } from 'react-redux'
import { decrement, increment } from './counterSlice'

export function Counter() {
    const count = useSelector((state) => state.counter.value) // 从state中获取count的值
    const dispatch = useDispatch() // 获取dispatch方法

    return (
        <div>
            <button onClick={() => dispatch(increment())}>
                Increment
            </button>
            <span>{count}</span>
            <button onClick={() => dispatch(decrement())}>
                Decrement
            </button>
        </div>
    )
}
```

或者用`connect`函数包裹组件，将state和dispatch映射到props中

```jsx
connect(mapStateToProps, mapDispatchToProps)(MyComponent)
```

```jsx
const mapStateToProps = (state) => {
    return {
        // prop : state.xxx  | 意思是将state中的某个数据映射到props中
        foo: state.bar
    }
}

const mapDispatchToProps = (dispatch) => { // 默认传递参数就是dispatch
    return {
        onClick: () => {
            dispatch({
                type: 'increatment'
            });
        }
    };
}

class Foo extends Component {
    constructor(props){
        super(props);
    }
    render () {
        return (
            <>
                <div>this.props.foo</div>
                <button onClick = {this.props.onClick}>点击increase</button>
            </>
        )
    }
}
Foo = connect(mapStateToProps, mapDispatchToProps)(Foo);
export default Foo;
```

## 同步react-router

### 安装

```bash
npm install --save connected-react-router
```

### 使用

1.创建一个以history作为参数，返回根reducer的函数

```jsx
// reducers.js
import { combineReducers } from 'redux'
import { connectRouter } from 'connected-react-router'

const createRootReducer = (history) => combineReducers({
  router: connectRouter(history),
  ... // rest of your reducers
})
export default createRootReducer;
```

2.创建一个history对象，将这个对象给上述的reducer和`createRootReducer`

```jsx
// configureStore.js
import { createBrowserHistory } from 'history'
import { applyMiddleware, compose, createStore } from 'redux'
import { routerMiddleware } from 'connected-react-router'
import createRootReducer from './reducers'

export const history = createBrowserHistory()

export default function configureStore(preloadedState) {
    const store = createStore(
        createRootReducer(history), // root reducer with router state
        preloadedState,
        compose(
            applyMiddleware(
                routerMiddleware(history), // for dispatching history actions
                // ...
            ),
        ),
    )

    return store
}
```

3.利用 ConnectedRouter组件，传递给他history对象作为prop，并将该组件作为 react-redux的Provider组件的子组件

```jsx
// index.js
import { Provider } from 'react-redux'
import { Route, Switch } from 'react-router' // react-router v4/v5
import { ConnectedRouter } from 'connected-react-router'
import configureStore, { history } from './configureStore'
const store = configureStore(/* 提供初始state */)

ReactDOM.render(
    <Provider store={store}>
        <ConnectedRouter history={history}> { /* place ConnectedRouter under Provider */ }
            <> 
            <Switch>
                <Route exact path="/" render={() => (<div>Match</div>)} />
                <Route render={() => (<div>Miss</div>)} />
            </Switch>
            </>
        </ConnectedRouter>
    </Provider>,
    document.getElementById('react-root')
)
```

### 编程式导航

connect-react-router也提供了路由跳转的方法，比如 push 和 replace，但是这些方法只是创建了action，需要dispatch这些方法产生的action

```jsx
import {push, replace} from "connected-react-router";

function XXX() {
    dispatch(push("/page1"))
    // ...
}
```

## 参考

[react学习笔记6-redux](https://blog.sakura-snow.com/post/react-node-6/)
[一篇文章总结redux、react-redux、redux-saga](https://juejin.cn/post/6844903846666321934)
[Redux中文文档](https://cn.redux.js.org/)
