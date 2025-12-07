---
title: React Hook
date: 2021-06-08 22:45:46
tags:
  - React
categories:
  - 前端
---
> 函数式组件只能使用props，Hook能够在函数式组件的情况下使用state、生命周期以及其他的React特性

为什么需要引入React Hook，可以查看官方文档：https://zh-hans.reactjs.org/docs/hooks-intro.html#motivation
<!--more-->
## 1. 注意事项

- 只能在函数内部的最外层调用 Hook，不要在循环、条件判断或者子函数中调用
- 只能在 React 的函数组件或自定义Hook中调用 Hook，不要在其他 JavaScript 函数中调用
- Hook 在 class 内部是**不**起作用的

## 2. useState

useState在组件内创建的内部state，React 会在重复渲染时保留这个 state，该函数的**第一个参数是初始值**，可以传入一个函数，此函数只在初始渲染中被调用。调用该函数会返回一对值：[当前状态，更新当前状态的函数]。

这个更新状态的函数**不会对state进行合并，而是直接替换**，可以传递一个回调函数，携带参数是上一次的state

```jsx
import React, { useState } from 'react'; 2:
function Example() {
    const [count, setCount] = useState(0); 5:
    return (
        <div>
            <p>You clicked {count} times</p>
            <button onClick={() => setCount(count + 1)}>
                Click me
            </button>
        </div>
    );
}
```

内部使用`Object.js`（浅）来比较新/旧`state`是否相等，在修改状态时，**传的状态值没有变化，则不重新渲染**

### 惰性初始值

下面的代码中，`initialState`是只在初始化时有其存在价值，但是如果真如下面一样写了，那么这个计算出`initialState`昂贵的操作在每次render都会执行。

```jsx
const initialState = someExpensiveComputation(props); // 这是一个耗时的操作
const [state, setState] = useState(initialState);
```

我们可以让`someExpensiveComputation` 运行在一个`useState`匿名函数参数下，该函数当且仅当初始化时被调用，从而优化性能。

```jsx
const [state, setState] = useState(() => {
    const initialState = someExpensiveComputation(props);
    return initialState;
});
```

## 2. useEffect

用来在函数式组件内使用class组件的生命周期函数，可以传递两个参数，第一个是执行回调函数，第二个是监听的变量数组。

### 使用方式

#### 1.只在第一次的componentDidMount执行

第二个参数为 []

```jsx
useEffect(()=>{
    // ...
}, [])
```

#### 2. 在第一次渲染和每次更新后执行

第二个参数为空

```jsx
useEffect(()=>{
    // ...
})
```

#### 3. 监听变量的变化执行

第二个参数是一个变量数组，只要有一个变量变化了就会执行

```jsx
useEffect(() => {
    // count改变才会执行
}, [count])
```

#### 4. 在componentWillUnmount中执行

第一个回调函数可以return一个函数，这个return的函数会在`componentWillUnmount`这个生命周期执行。

```jsx
useEffect(() => {
    console.log('use effect...', count)
    const timer = setInterval(() => setCount(count +1), 1000)
    return () => clearInterval(timer)
})
```

## 3. useRef

`useRef` 返回一个可变的 ref 对象，其 `.current` 属性被初始化为传入的参数（`initialValue`）。返回的 ref 对象在组件的整个生命周期内保持不变。

当 ref 对象内容发生变化时，`useRef` 并*不会*通知你。变更 `.current` 属性不会引发组件重新渲染。

**和createRef的区别**：createRef 每次渲染都会返回一个新的引用，而 useRef 每次都会返回相同的引用（可以解决每次渲染引用不同的useState，导致状态异常的bug）

```jsx
function App () {
    const [renderIndex, setRenderIndex] = useState(1);
    const refFromUseRef = useRef();
    const refFromCreateRef = createRef();
    if (!refFromUseRef.current) {
        console.log("useRef")
        refFromUseRef.current = renderIndex;
    }
    if (!refFromCreateRef.current) {
        console.log("createRef")
        refFromCreateRef.current = renderIndex;
    }
    return (
        <div className="App">
            Current render index: {renderIndex}
            <p>refFromUseRef value: {refFromUseRef.current}</p>
            <p>refFromCreateRef value: {refFromCreateRef.current}</p>
            <button onClick={() => setRenderIndex(prev => prev + 1)}>
                Cause re-render
            </button>
        </div>
    );
}
```

![image-20210606235123989](D:\OneDrive - mail2.gdut.edu.cn\typora_img\React Hook\image-20210606235123989.png)

## 4. useImperativeHandle

`useImperativeHandle`可以让你在使用 ref 时，自定义暴露给父组件的实例值，不能让父组件想干嘛就干嘛

`useImperativeHandle` 应当与 [`forwardRef`](https://zh-hans.reactjs.org/docs/react-api.html#reactforwardref) 一起使用：

```jsx
function Child (props, parentRef) {
    // 子组件内部自己创建 ref
    let focusRef = useRef();
    let inputRef = useRef();
    useImperativeHandle(parentRef, () => {
        // 这个函数会返回一个对象
        // 该对象会作为父组件 current 属性的值
        // 通过这种方式，父组件可以使用操作子组件中的多个 ref
        return {
            focusRef,
            inputRef,
            name: '计数器',
            focus () {
                focusRef.current.focus();
            },
            changeText (text) {
                inputRef.current.value = text;
            }
        }
    });
    return (
        <React.Fragment>
            <input ref={focusRef}/>
            <input ref={inputRef}/>
        </React.Fragment>
    )
}

Child = forwardRef(Child);

function Parent () {
    const parentRef = useRef();//{current:''}
    function getFocus () {
        parentRef.current.focus();
        // 因为子组件中没有定义这个属性，实现了保护，所以这里的代码无效
        // parentRef.current.addNumber(666);
        parentRef.current.changeText('777');
        console.log(parentRef.current.name);
    }

    return (
        <React.Fragment>
            <Child ref={parentRef}/>
            <button onClick={getFocus}>获得焦点</button>
        </React.Fragment>
    )
}
```

## 5.useMemo

> 父组件的state改变，子组件也会随之重新render，即使子组件内部state没有改变，我们可以用useMemo来进行性能优化

参数列表

- 回调函数，return出来的值作为useMemo的返回值
- 依赖项数组，只要有一个变量改变，就会重新执行回调，但不会触发渲染，如果**没有提供依赖数组，则`useMemo` 在每次渲染时都会计算新的值**

`useMemo`返回一个 [memoized](https://en.wikipedia.org/wiki/Memoization) 值，它仅会在会在某个依赖项改变时才重新计算 memoized 值。

```jsx
const Child = memo(({data}) => {
    console.log('child render...', data.name)
    return (
        <div>
            <div>child</div>
            <div>{data.name}</div>
        </div>
    );
})

const Hook = () => {
    console.log('Hook render...')
    const [count, setCount] = useState(0)
    const [name, setName] = useState('rose')

    const data = useMemo(() => {
        return {
            name
        }
    }, [name])

    return(
        <div>
            <div>
                {count}
            </div>
            <button onClick={() => setCount(count + 1)}>update count </button>
            <Child data={data}/>
        </div>
    )
}
```

## 6. useCallback

> `useCallback`与`useMemo`的差别是，前者是缓存函数，后者是缓存值

```jsx
const onChange = useCallback((e) => {
    setText(e.target.value)
}, [])
```

`useCallback(fn, deps)`相当于`useMemo(() => fn, deps)`

## 7. useContext

接收一个 context 对象（由`React.createContext` 所创建）并返回该 context 的当前值，当前的 context 值由上层组件中距离当前组件最近的 `<MyContext.Provider>` 的 `value` prop 决定

```jsx
const TextContext = React.createContext();

function App() {
  return (
    <TextContext.Provider value={666}>
      <Toolbar />
    </TextContext.Provider>
  );
}

function Toolbar(props) {
  return (
    <div>
      <xxxButton />
    </div>
  );
}

function xxxButton() {
  const text = useContext(ThemeContext);
  return (
    <button style={{ color: text }}>
      I am styled by theme context!
    </button>
  );
}
```

当组件上层最近的 `<MyContext.Provider>` 更新时，该 Hook 会触发重渲染，并使用最新传递给 `MyContext` provider 的 context `value` 值。即使祖先使用 [`React.memo`](https://zh-hans.reactjs.org/docs/react-api.html#reactmemo) 或 [`shouldComponentUpdate`](https://zh-hans.reactjs.org/docs/react-component.html#shouldcomponentupdate)，也会在组件本身使用 `useContext` 时重新渲染

## 8. useLayoutEffect

- `useLayoutEffect`会在浏览器 layout 之后，painting 之前执行

- 其函数签名与 useEffect 相同，但它会在所有的 DOM 变更之后**同步**调用 effect

- **可以使用它来读取 DOM 布局并同步触发重渲染**

- **尽可能使用标准的 useEffect 以避免阻塞视图更新**

```jsx
function LayoutEffect () {
    const [color, setColor] = useState('red');
    useLayoutEffect(() => {
        alert(color);
    });
    useEffect(() => {
        console.log('color', color);
    });
    return (
        <React.Fragment>
            <div id="myDiv" style={{ background: color }}>颜色</div>
            <button onClick={() => setColor('red')}>红</button>
            <button onClick={() => setColor('yellow')}>黄</button>
            <button onClick={() => setColor('blue')}>蓝</button>
        </React.Fragment>
    );
}
```

 先alert出red，再渲染出页面，阻塞渲染

## 9. 自定义Hook

其实就是写一个函数内部调用其他Hook，每次使用自定义Hook时，所有的state和副作用都是完全隔离的

例如：我们可以对useRef封装成自定义Hook来获取上一次的值

```jsx
const usePrevious = state => {
    const ref = useRef();

    useEffect(() => {
        ref.current = state;
    })

    return ref.current;
}
```

```jsx
function App () {
    const [count, setCount] = useState(0);
    const prevCount = usePrevious(count);

    return (
        <div className="App">
            <button onClick={() => setCount(count + 1)}>
                Cause re-render
            </button>
            <p> {count} {prevCount}</p>
        </div>
    );
}
```

每次点击button，prevCount都是上一次count的值

## 易错点

### 1. 为什么我的count没有更新

```jsx
function ErrorDemo() {
    const [count, setCount] = useState(0);
    const dom = useRef(null);
    useEffect(() => {
        dom.current.addEventListener('click', () => setCount(count + 1));
    }, []);
    return <div ref={dom}>{count}</div>;
}
```

上面这段代码，用户点击div，count只会加到1，后面就不会再加了。

原因是：每次 `count` 都是重新声明的变量，指向一个全新的数据；每次的 `setCount` 虽然是重新声明的，但指向的是同一个引用。

### 解决方式一：函数式更新

用回调函数的形式，`() => setCount(prevCount => ++prevCount)`，来消除对外部count的引用。

### 解决方式二：重新绑定事件

```jsx
useEffect(() => {
    dom.current.addEventListener('click', () => setCount(count + 1));
    return () => dom.current.removeEventListener('click', () => setCount(count + 1));
}, [count]); // 在这里对count进行监听，每次改变都会重新绑定事件
return <div>{count}</div>;
```

### 解决方式三：用ref重新获取引用

```jsx
const dom = useRef(null);
const countRef = useRef(count);
useEffect(() => {
    dom.current.addEventListener('click', () => {
        countRef.current++;
        setCount(countRef.current);
    });
}, []);
```

## 参考

[写React Hooks前必读](https://juejin.cn/post/6844904090032406536)
[终于搞懂 React Hooks了！！！！！](https://juejin.cn/post/6844904072168865800)

