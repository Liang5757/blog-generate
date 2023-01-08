---
title: 尤雨溪frontend Master课程笔记
date: 2021-02-18 02:46:31
tags:
  - vue
categories:
  - 前端
---
## 1.响应式

### 目的

实现一个神奇的函数auto，会在`state.count`改变后，自动运行里面的函数

```js
autoRun(() => {
    document.getElementById("app").innerText = state.count;
})

state.count++; // 重新执行autoRun内的函数
```
<!--more-->
### 第一步 getter和setter

需要能监听到对象内属性的改变

#### 实现效果

```js
const obj = { foo: 123 }
observer(obj);
obj.foo // 需要打印: 'getting key "foo": 123'
obj.foo = 234 // 需要打印: 'setting key "foo" to 234'
obj.foo // 需要打印: 'getting key "foo": 234'
```

#### 实现方式

```js
function observer (obj) {
  Object.keys(obj).forEach(key => {
    // 保存属性初始值
    let internalValue = obj[key]
    Object.defineProperty(obj, key, {
      get () {
        console.log(`getting key "${key}": ${internalValue}`)
        return internalValue
      },
      set (newValue) {
        console.log(`setting key "${key}" to: ${newValue}`)
        internalValue = newValue
      }
    })
  })
}
```

### 第二步 依赖收集 Dep

#### 实现效果

即一个发布订阅模式

```js
const dep = new Dep()

autoRun(() => {
  dep.depend()
  console.log('updated')
})
// 打印: "updated"

dep.notify()
// 打印: "updated"
```

#### 实现方式

```js
class Dep {
  constructor() {
    this.subs = []
  };
  
  depend() { // 订阅函数
    if (activeUpdate) {
      this.subs.push(activeUpdate); // 把全局变量activeUpdate存的函数放入订阅者列表
    }
  }
  
  notify() { // 发布函数
    this.subs.forEach(sub => sub());
  }
}


let activeUpdate = null // 放置依赖函数

function autoRun (update) {
  const wrappedUpdate = () => {
    activeUpdate = wrappedUpdate // 把wrappedUpdate存起来
    update() // 在update内部调用dep.depend()收集依赖
    activeUpdate = null
  }
  wrappedUpdate()
}
```

### 第三步 整合一、二

```js
class Dep {
  constructor() {
    this.subs = []
  };
  
  depend() {
    if (activeUpdate) {
      this.subs.push(activeUpdate);
    }
  }
  
  notify() {
    this.subs.forEach(sub => sub());
  }
}

export class Observer {
  constructor(obj) {
    this.dep = new Dep();
    this.walk(obj);
  }
  
  walk(obj) {
    Object.keys(obj).forEach(key => {
      this.defineReactive(obj, key);
    })
  }
  
  defineReactive(obj, key) {
    let that = this;
    let internalValue = obj[key];
    Object.defineProperty(obj, key, {
      enumerable: false,
      configurable: false,
      get() {
        that.dep.depend();
        return internalValue;
      },
      set(newVal) {
        const isChanged = internalValue !== newVal;
        if (isChanged) {
          internalValue = newVal;
          that.dep.notify();
        }
      }
    })
  }
}

let activeUpdate = null;

export function autoRun (update) {
  const wrappedUpdate = () => {
    activeUpdate = wrappedUpdate // 把wrappedUpdate存起来
    update() // 在update内部调用dep.depend()收集依赖
    activeUpdate = null
  }
  wrappedUpdate()
}
```

## 2. 虚拟DOM

### 2.1虚拟DOM和真实的DOM的差异

1.资源消耗问题

使用javascript操作真实DOM是非常消耗资源的，虽然很多浏览器做了优化但是效果不大。你看到虚拟DOM是一个纯javascript对象。而DOM节点有70＋个属性，继承层级有6，7层（文本节点6层，元素节点7层）,访问一个属性，可能会追溯几重原型链。

2.执行效率问题

如果你要修改一个真实DOM，一般调用`innerHTML`方法，那浏览器会把旧的节点移除再添加新的节点，但是在虚拟DOM中，只需要修改一个对象的属性，再把虚拟DOM渲染到真实DOM上。很多人会误解虚拟DOM比真实DOM速度快，其实虚拟DOM只是把DOM变更的逻辑提取出来，使用javascript计算差异，减少了操作真实DOM的次数，只在最后一次才操作真实DOM，所以如果你的应用有复杂的DOM变更操作，虚拟DOM会比较快。

3.虚拟DOM还有其他好处

其实虚拟DOM还可以应用在其他地方，因为他们只是抽象节点，可以把它编译成其他平台，例如android、ios。市面上利用形同架构模式的应用有React Native，Weeks，Native script，就是利用虚拟DOM的特点实现的。

### 2.2 虚拟DOM在线查看

使用Vue Template Explorer可以查看Vue是如何转换虚拟DOM的。

[访问地址](https://template-explorer.vuejs.org/)

## 3.template和jsx对比

**模版的优势**：模版是一种更静态更具有约束的表现形态，它可以避免发明新语法，任何可以解析HTML的引擎都可以使用它，迁移成本更低；另外最重要的是**静态模版可以在编译进行比较多的优化**，而动态语言就没法实现了。

**jsx的优势**：更灵活，任何的js代码都可以放在jsx中执行实现你想要的效果，但是也**由于他的灵活性导致在编译阶段优化比较困难，只能通过开发者自己优化**。

## 4. 函数组件

函数组件就是不包含state和props的组件，就像它的名字一样，你可以理解为他就是一个函数，在Vue中声明一个函数组件代码如下：

```js
const foo = {
	functional: true,
    render: h => h('div', 'foo')
}
```

### 特点

1. 组件不支持实例化。
2. 优化更优，因为在Vue中它的渲染函数比父级组件更早被调用，但是他并不会占用很多资源，因为它没有保存数据和属性，所以它常用于优化一个有很多节点的组件。
3. 容易扩展，如果你的组件只是用来接收 prop然后显示数据，或者一个没有状态的按钮，建议使用函数组件。
4. 函数组件没有this，获取prop可以通过render函数的第二参数得到`render(h, context)`

```js
Vue.component('example', {
    functional: true, // 声明是函数组件
    // 因为函数组件没有this,可以通过render第二参数获取相关信息
    render(h, { props: { tags } }) {
        // context.slots() 通过slots方法获取子节点
        // context.children 获取子组件
        // context.parent 父组件，因为函数组件实挂载到根节点上，也就是<div id="app"></div>
        // context.props 组件属性，这里得到tags属性
        // return h('div', this.tags.map((tag, i) => h(tag, i)))
        // 通过函数组件实现标签动态渲染
        return h('div', tags.map((tag, i) => h(tag, i)))
    }
})
```

## 5.HOC 高阶组件

> 高阶组件是一个函数，接收一个组件，然后返回一个新的组件，类似装饰者模式

这里不展开说了，大概列一下写法，下面模拟了一个图片骨架

```js
  // mock API
  function fetchURL (username, cb) {
    setTimeout(() => {
      // hard coded, bonus: exercise: make it fetch from gravatar!
      cb('https://avatars3.githubusercontent.com/u/6128107?v=4&s=200')
    }, 500)
  }

  const Avatar = {
    props: ['src'],
    template: `<img :src="src">`
  }

  function withAvatarURL (InnerComponent) {
    return {
      props: {
        attrs: this.$attrs, // 2.4 only
        username: String
      },
      data () {
        return {
          url: 'http://via.placeholder.com/200x200'
        }
      },
      created () {
        fetchURL(this.username, (url) => { this.url = url })
      },
      render (h) {
        return h(InnerComponent, { props: { src: this.url } })
      }
    }
  }

  const SmartAvatar = withAvatarURL(Avatar)

  new Vue({
    el: '#app',
    components: { SmartAvatar }
  })
```

1. **重用性**：因为minxin对原组件具有侵入性，这会导致原来组件的可重用性降低，而高阶组件不会，高阶组件对原组件只是一个调用关系，并没有修改原来组件任何内容。
2. **可测试性**：因为高阶组件只是一个嵌套关系，在组件测试的时候，可以单独的测试原始组件和高阶组件。
3. **层级问题**：高阶组件也有他的弊端，如果你高阶组件嵌套层级太深，会导致出错的时候调试困难的问题，所以到底使用高阶组件和minxin需要看实际场景。

## 6. 路由

实现根据路由匹配显示组件，并路由匹配参数

```js
// 组件
const Foo = {
    props: ['id'],
    template: `<div>foo with id: {{ id }}</div>`
}
const Bar = { template: `<div>bar</div>` }
const NotFound = { template: `<div>not found!</div>` }

// 路由表
const routeTable = {
    '/foo/:id': Foo,
    '/bar': Bar
}

// 将路由表的键通过 path-to-regexp库 进行正则封装
// 下面这个数组储存：组件、正则对象、匹配的name
const compiledRoutes = []
Object.keys(routeTable).forEach(key => {
    const dynamicSegments = []
    const regex = pathToRegexp(key, dynamicSegments)
    const component = routeTable[key]
    compiledRoutes.push({
        component,
        regex,
        dynamicSegments
    })
})

// 监听hashchange，将改变的路由赋值给url
window.addEventListener('hashchange', () => {
    app.url = window.location.hash.slice(1)
})

const app = new Vue({
    el: '#app',
    data: {
        url: window.location.hash.slice(1)
    },
    render (h) {
        const path = '/' + this.url

        let componentToRender // 要渲染的组件
        let props = {} // 路由匹配到的值

        compiledRoutes.some(route => {
            const match = route.regex.exec(path) // 执行匹配
            componentToRender = NotFound
            if (match) {
                componentToRender = route.component
                // 设置参数
                route.dynamicSegments.forEach((segment, index) => {
                    props[segment.name] = match[index + 1]
                })
                return true
            }
        })

        return h('div', [
            h(componentToRender, { props }),
            h('a', { attrs: { href: '#foo/123' }}, 'foo 123'),
            ' | ',
            h('a', { attrs: { href: '#foo/234' }}, 'foo 234'),
            ' | ',
            h('a', { attrs: { href: '#bar' }}, 'bar'),
            ' | ',
            h('a', { attrs: { href: '#garbage' }}, 'garbage')
        ])
    }
})
```

