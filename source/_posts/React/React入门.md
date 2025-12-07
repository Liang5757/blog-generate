---
title: React从入门到真香
date: 2021-05-27 23:41:33
tags:
  - React
categories:
  - 前端
---
## 1. 设计思想

1. [React 设计思想](https://github.com/react-guide/react-basic)
2. [React的设计哲学 - 简单之美](http://www.infoq.com/cn/articles/react-art-of-simplity/)
3. [颠覆式前端UI开发框架:React](http://www.infoq.com/cn/articles/subversion-front-end-ui-development-framework-react/)
<!--more-->
## 2. 安装

### 2.1 CDN的方式引入

```html
<script src="https://unpkg.com/react@17/umd/react.development.js" crossorigin></script>
<script src="https://unpkg.com/react-dom@17/umd/react-dom.development.js" crossorigin></script>
<script src="https://unpkg.com/babel-standalone@6/babel.min.js"></script>
```

- react.js：React核心库
- react-dom.js：提供操作 DOM 的 react 扩展库
- babel.min.js：解析 JSX 语法代码转为 JS 代码的库

### 2.2 create-react-app

安装`create-react-app`

```bash
npm isntall -g create-react-app
```

创建项目

```bash
create-react-app my-app
```

## 3. JSX

jsx 允许在模板中用`{}`插入JavaScript表达式，如果`{}`中的变量是数组，则会展开这个数组的所有成员

```js
var arr = [
  <h1>Hello world!</h1>,
  <h2>React is awesome</h2>,
];
ReactDOM.render(
  <div>{arr}</div>,
  document.getElementById('example')
);

var names = ['Alice', 'Emily', 'Kate'];

ReactDOM.render(
  <div>
  {
    names.map(function (name) { // 列表渲染
      return <div>Hello, {name}!</div>
    })
  }
  </div>,
  document.getElementById('example2')
);
```

`ReactDOM.render`第一个参数是渲染的组件，第二个参数是挂载实例的位置。

- className替换class
- 内联样式——style={{color}}
- 只能含有一个最外层标签

对于最后一个问题，可以使用React.Fragment组件，它能够在不额外创建 DOM 元素的情况下，让render返回多个元素

```jsx
render() {
  return (
    <React.Fragment>
      Some text.
      <h2>A heading</h2>
    </React.Fragment>
  );
}
```

## 4.组件

### 4.1 类组件（有状态）

> 组件类的第一个名字必须大写

通过继承React.Component来实现类组件，需要实现一个render方法，该方法返回一个模板

```jsx
class Clock React.Component {
    render () {
        return (
            <div onClick={this.clickFunc.bind(this)}> // 将clickFunc内部this指向组件实例
                <h1>Hello, world!</h1>
                <h2>It is {this.props.date.toLocaleTimeString()}</h2>
            </div>
        )
    }
}
```

 通过React.createClass生成一个组件类

```jsx
var Clock = React.createClass({
    render: function() {
        return (
            <div>
                <h1>Hello, world!</h1>
                <h2>It is {this.props.date.toLocaleTimeString()}</h2>
            </div>
        )
    }
});
```

两种方式的区别是，前者不会自动绑定this

### 4.2 函数式组件（无状态）

```jsx
function Person (props) {
    const {name, age, sex} = props;
    return {
        <ul>
            <li>姓名：{name}</li>
            <li>性别：{sex}</li>
            <li>年龄：{age}</li>
        </ul>
    }
}
```

## 5. State

### 5.1 state初始化

```jsx
class Clock React.Component {
  // 第一种 直接在实例上定义state属性
  state = {
      value: 6,
  }
    
  // 第二种 在构造器中设置state
  constructor () {
      this.state = {
          value: 6
      }
  }
}
```

### 5.2 state修改

**直接修改数据是不会触发视图更新**的，只有使用`setState`来修改数据，会重新触发组件的render函数

setState第一个参数就是修改后的state对象，可以修改某一个值，**setState是浅合并**。

如果setState的第一个参数不是一个对象而是一个函数，这个函数在执行时会通过参数被传入prevState，也就是之前的状态，而返回值就会和state进行合并

```js
this.setState((prevState : any) => {
    return {
        num : prevState.num + 1
    }
}, () => {
    console.log(this.state.num)
})
```

但是setState有两个问题

- setState对状态的修改，可能是异步执行的（如果改变状态的代码处于某个HTML元素的事件中，则其是异步的，否则是同步）
- React会对异步的setState进行优化，将多次setState进行合并（将多次状态改变完成后，再统一对state进行改变，然后触发render）

下面给出第一个问题的例子

```jsx
class Comp extends React.Component {
    state = {
        num: 0
    }

    handleClick = () => {
        this.setState({
            num: this.state.num + 1
        })
        console.log(this.state.num) // 第一下点击输出 0
    }

    render () {
        return (
            <React.Fragment>
                <p>{this.state.num}</p>
                <button onClick={this.handleClick}>点我</button>
            </React.Fragment>
        )
    }
}
```

在第一下点击按钮的时候，console.log输出 0，说明setState在DOM事件中是异步执行的。

**解决方式**：setState有第二个参数，来放置render执行后接着执行的回调函数

## 6. Props

### 6.1 传递参数

父组件在使用子组件的时候，通过给子组件元素设置属性的方式传递参数给子组件

```jsx
// 父组件设置属性给子组件传参，这里属性名用value2用作区分
class Board extends React.Component {
    render () {
        return (
            <Square value2={0}/>
        );
    }
}

// 类组件 通过 this.props.[属性] 来获取参数
class Square extends React.Component {
    render() {
        return (
            <button className="square">
                {this.props.value2}
            </button>
        );
    }
}

// 函数式组件通过 props.[属性] 来获取参数
function Square (props) {
    return (
        <button className="square">
            {props.value2}
        </button>
    )
}
```

### 6.2 props传递事件

通过this.props.[事件名]调用

```jsx
class Board extends React.Component {
    render () {
        return (
            <Square click={() => {alert("click event")}}/>
        );
    }
}

class Square extends React.Component {
    render() {
        return (
            <button onClick={this.props.click} className="square">
              	button
            </button>
        );
    }
}
```

### 6.3 props校验

1.需要引入一个库prop-types.js

- cdn

```html
<script src="https://unpkg.com/prop-types@15.6/prop-types.js"></script>
```

- npm

```bash
npm install --save prop-types 
```

2.使用方式

在组件上定义 PropTypes对象，键就是props的名，值就是限制。

```js
class Square extends React.Component {
    render() {
        // ... do things with the props
    }

    // 方式一
    static propTypes = {
        value1: PropTypes.number.isRequired,
        value2: PropTypes.string
    }

	// 指定props默认值
	static defaultProps = {
        value1: 666,
        value2: "6",
    }
}

// 方式二
Square.propTypes = {
    value1: PropTypes.number.isRequired,
    value2: PropTypes.string
}
```

更多使用方式参考：https://github.com/facebook/prop-types

### 6.4 this.props.children

> 获取组件所有子节点

```jsx
class NotesList extends React.Component {
    render() {
        return (
            <ol>
                {
                    React.Children.map(this.props.children, function (child) {
                        return <li>{child}</li>;
                    })
                }
            </ol>
        );
    }
}

ReactDOM.render(
    <NotesList>
        <span>hello</span>
        <span>world</span>
    </NotesList>,
    document.getElementById('example')
);
```

- 没有子节点：undefined
- 一个子节点：object
- 多个子节点：array

可以用`React.Children.map`来遍历子节点，则不需要担心子元素个数

### 6.5 props传递组件

React组件本质就是对象，当作props像其他数据一样传递也是可以的

##  7. React生命周期

> 感觉和vue差不多，直接简单带过把

#### componentWillMount()

组件即将被渲染到页面之前触发，此时可以进行开启定时器、向服务器发送请求等操作

#### componentDidMount()

组件已经被渲染到页面中后触发，可以通过`this.getDOMNode()`来进行访问DOM。

#### componentWillReceiveProps()

在组件接收到一个新的 prop (更新后)时被调用。这个方法在初始化`render`时不会被调用。

#### shouldComponentUpdate()

返回一个布尔值。在组件接收到新的`props`或者`state`时被调用。在初始化时或者使用`forceUpdate`时不被调用。

```jsx
// 该钩子函数可以接收到两个参数，新的属性和状态，返回true/false来控制组件是否需要更新。
shouldComponentUpdate(newProps, newState) {
    if (newProps.number < 5) return true;
    return false
}
```

> 一个React项目需要更新一个小组件时，很可能需要父组件更新自己的状态。而一个**父组件的重新更新会造成它旗下所有的子组件重新执行render()方法（即使没有使用父组件的state）**，形成新的虚拟DOM，再用diff算法对新旧虚拟DOM进行结构和属性的比较，决定组件是否需要重新渲染，还可以使用下面说了PureComponent

#### componentWillUpdate()

在组件接收到新的`props`或者`state`但还没有`render`时被调用。在初始化时不会被调用。

#### componentDidUpdate()

在组件完成更新后立即调用。在初始化时不会被调用。

#### componentWillUnMount()

组件被销毁时触发。这里我们可以进行一些清理操作，例如清理定时器，取消Redux的订阅事件等等。

#### getDerivedStateFromError()

这个生命周期方法在**ErrorBoundary**类中使用。实际上，如果使用这个生命周期方法，任何类都会变成`ErrorBoundary`。这用于在组件树中出现错误时呈现回退UI，而不是在屏幕上显示一些奇怪的错误。

#### componentDidCatch()

这个生命周期方法在**ErrorBoundary**类中使用。实际上，如果使用这个生命周期方法，任何类都会变成`ErrorBoundary`。这用于在组件树中出现错误时记录错误。

## 8. Ref

用来访问DOM元素或render中的react元素，Ref的使用规则如下

1. ref作用于内置的html组件时，得到的将是真实的dom对象
2. ref作用于类组件时，得到的将是类的实例
3. ref不能作用于函数组件（因为没有实例），但是函数组件内部可以

### 8.1 创建ref

ref的可选值为

1. 字符串（不建议）

	原因：[react ref注意事项](http://www.qiutianaimeili.com/html/page/2020/04/20204277uz3udr631a.html)

2. 回调函数

	触发时机：ref中的回调函数会在对应的**普通**组件（或元素）`componentDidMount`，`ComponentDidUpdate`之前，或者`componentWillUnmount`之后执行

	**注意：**如果 `ref` 回调函数是以**内联函数的方式定义**的，在更新过程中它会被**执行两次**，第一次传入参数 `null`，然后第二次会传入参数 DOM 元素。这是因为在**每次渲染时会创建一个新的函数实例**，所以 React 清空旧的 ref 并且设置新的

3. createRef

```jsx
class MyComponent extends React.Component {
    constructor(props) {
        super(props);
        this.textInput = React.createRef();
    }
    focusTextInput() {
        // 注意：我们通过 "current" 来访问 DOM 节点
        this.textInput.current.focus();
    }
    render() {
        return (
            <React.Fragment>
                <input type="text" ref={this.textInput} />
                <input type="button" value="Focus the text input" onClick={this.focusTextInput.bind(this)}/>
            </React.Fragment>
        );
    }
}
```

### 8.2 转发ref

> 用于获取组件内部的某个元素，有些高度复用的基础组件不可避免的需要在父组件获取，用以管理焦点等

`React.forwardRef`用以获取传递给它的`ref`，然后转发到渲染它的DOM。

对于**函数式组件**，`React.forwardRef`直接包裹函数就可以接收到ref

```jsx
const FancyButton = React.forwardRef((props, ref) => (
  <button ref={ref} className="FancyButton">
    {props.children}
  </button>
));

// 你可以直接获取 DOM button 的 ref：
const ref = React.createRef();
<FancyButton ref={ref}>Click me!</FancyButton>;
```

使用 `FancyButton` 的组件可以获取底层 DOM 节点 `button` 的 ref 

而对于**类组件**，需要使用HOC的形式，用`React.forwardRef`包裹返回的函数式组件

```jsx
function logProps(Component) {
    class LogProps extends React.Component {
        render() {
            const {forwardedRef, ...rest} = this.props;

            // 将自定义的 prop 属性 “forwardedRef” 定义为 ref
            return <input ref={forwardedRef} {...rest} />;
        }
    }

    // 注意 React.forwardRef 回调的第二个参数 “ref”。
    // 我们可以将其作为常规 prop 属性传递给 LogProps，例如 “forwardedRef”
    // 然后它就可以被挂载到被 LogProps 包裹的子组件上。
    return React.forwardRef((props, ref) => {
        return <LogProps {...props} forwardedRef={ref} />;
    });
}
```

用`React.forwardRef`包裹的组件在**DevTools中显示为"ForwardRef"**，可以把包裹的函数用普通函数的形式命名，DevTools也将包含其名称（例如 “*ForwardRef(myFunction)*”），也可以**设置函数的displayName属性**来设置DevTools中显示的名字

## 9.Context

> 跨组件的通信方式，等同Vue的provide、inject

### 使用

1.创建Context容器对象

```jsx
// 只有当组件所处的树中没有匹配到 Provider 时，其 defaultValue 参数才会生效
const xxxContext = React.createContext(defaultValue) 
```

2.渲染子组时，外面包裹xxxContext.Provider，通过value属性给后代组件传递数据

```jsx
<xxxContext.Provider value={数据}>
    <子组件></子组件>
</xxxContext.Provider>
```

ps：如果要传多个数据，需要套多层

3.后代组件获取数据

```jsx
// 第一种：仅适用于类组件
static contextType = xxxContext // 声明接受context
this.context; // 读取context中的value数据

// 第二种：函数组件与类组件都可以
<xxxContext.Consumer>
    {value}
</xxxContext.Consumer>
```

**不建议使用该api，他提高了组件复用的难度**

## 10.PureComponent

> 父组件setState会触发子组件render，PureComponent通过prop和state的**浅比较**来实现shouldComponentUpdate，算是一种语法糖，帮我们应该在shouldComponentUpdate中应该手动比较的给做了

如果是浅层state或prop没改变，那么不会触发视图更新，书写方式如下

```jsx
class IndexPage extends PureComponent {
	// ...
}
```

**由于是浅比较，所以直接修改对象内部的值是无法更新视图的**

在函数式组件则是用`React.memo`包裹函数式组件来做到PureComponent的效果

## 参考

[React学习资源汇总](https://github.com/tsrot/study-notes/blob/master/React%E5%AD%A6%E4%B9%A0%E8%B5%84%E6%BA%90%E6%B1%87%E6%80%BB.md)
[React 入门实例教程](http://www.ruanyifeng.com/blog/2015/03/react.html)
[react学习笔记2-react基本使用](https://blog.sakura-snow.com/post/react-node-2/)
[react学习笔记3-react其他使用技巧](https://blog.sakura-snow.com/post/react-node-3/)
[图解ES6中的React生命周期](https://juejin.cn/post/6844903510538977287)
[你要的 React 面试知识点，都在这了](https://juejin.cn/post/6844903857135304718#heading-25)

