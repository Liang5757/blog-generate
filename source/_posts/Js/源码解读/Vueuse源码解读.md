---
title: Vueuse源码解读
date: 2023-09-10 16:53:55
tags:
  - js
categories:
  - 源码解读
---

> 本文不会放api的用法，建议先看看是怎么用的
> 
> 写本篇文章时间间隔较长，所以代码版本不一

## 项目架构

采用monorepo的形式，项目目录下有多个子项目，下面放了资料链接和几处用法，其他本文不多赘述。

[现代前端工程为什么越来越离不开 Monorepo?](https://juejin.cn/post/6944877410827370504)
[为什么使用pnpm可以光速建立好用的monorepo（比yarn/lerna效率高）](https://www.codeleading.com/article/46915806308/)
[pnpm workspace文档](https://pnpm.io/zh/workspaces)

<!--more-->

为了使**所有子项目都使用同一个依赖版本**，使用[pnpm.overrides](https://pnpm.io/zh/package_json#pnpmoverrides)配置，如下

```json
{
  "pnpm": {
    "overrides": {
      "vue-demi": "0.12.1", // 在postinstall钩子中根据当前环境的vue版本，获取兼容vue3和vue2的api
      "vite": "^2.6.7"
    }
  }
}
```

在正常配置下，如果直接`pnpm install`将会在每个子项目的`node_modules`有同样的依赖，但是所有子项目相同的依赖其实只需要提取出来放在根目录的`node_modules`下即可，可配置如下

```json
  "peerDependencies": {
    "@vue/composition-api": "^1.1.0",
    "vue": "^2.6.0 || ^3.2.0"
  },
```

详情请看如下链接

[探讨npm依赖管理之peerDependencies](https://www.cnblogs.com/wonyun/p/9692476.html)
[pnpm monorepo之多组件实例和peerDependencies困境回溯](https://blog.csdn.net/qq_21567385/article/details/121088506)
[如何处理 peers](https://pnpm.io/zh/how-peers-are-resolved)

## 前置知识

### unref - ref的反操作

- 如果传入一个ref，返回其值
- 否则原样放回

```typescript
// 源码实现
function unref<T>(ref: T | Ref<T>): T {
  return isRef(ref) ? ref.value : ref
}
```

 使用例子：实现一个响应式的add函数，可以传入**ref或值**

```typescript
function add(
  a: Ref<number> | number,
  b: Ref<number> | number
) {
  return computed(() => unref(a) + unref(b))
}
```

### MaybeRef类型

vueuse大量使用`MaybeRef`来支持**可选择性的响应式参数**

```typescript
type MaybeRef<T> = Ref<T> | T
```

上面的加法函数就可以简写为

```typescript
function add(a: MaybeRef<number>, b: MaybeRef<number>) {
  return computed(() => unref(a) + unref(b))
}
```

### Effect作用域API

Vue官方文档：[Effect 作用域 API](https://v3.cn.vuejs.org/api/effect-scope.html#effectscope)

**动机**

在Vue的setup中，响应式effect会在初始化的时候被收集，在实例被卸载的时候，响应式effect就会自动被取消了，但是在我们在组件外写一个独立的包（就如vueuse）时，我们该如何取消computed & watch的响应式依赖呢？vue3.2提出了`effectScope`，接下来介绍相关的api

#### effectScope

利用`effectScope`创建一个作用域对象，如下面接口定义所示，`run`接受一个函数，这个作用域对象会自动捕获函数内部的响应式effect (例如计算属性或侦听器)

**类型**

```ts
function effectScope(detached?: boolean): EffectScope

interface EffectScope {
  run<T>(fn: () => T): T | undefined // 如果这个域不活跃则为 undefined
  stop(): void
}
```

**示例**

```js
const scope = effectScope()

scope.run(() => {
  const doubled = computed(() => counter.value * 2)
  watch(doubled, () => console.log(doubled.value))
  watchEffect(() => console.log('Count: ', doubled.value))
})

// 丢弃(dispose) 该作用域内的所有 effect
scope.stop()
```

#### getCurrentScope

如果有，则返回当前活跃的 [effect 作用域](https://v3.cn.vuejs.org/api/effect-scope.html#effectscope)。

**类型**

```ts
function getCurrentScope(): EffectScope | undefined 
```

#### onScopeDispose

在当前活跃的 [effect 作用域](https://v3.cn.vuejs.org/api/effect-scope.html#effectscope)上注册一个处理回调。该回调会在相关的 effect 作用域结束之后被调用，在VCA（Vue Composition API）函数中可用作`onUnmounted`的非组件替代品，区别在于其工作在scope中而不是组件实例

```ts
// 当前 effectScope 结束后调用
onScopeDispose(() => {
  window.removeEventListener('mousemove', handler)
})
```

那么如此抽象的`effectScope`在尤大降低心智负担的主张下为什么要被提出呢，下面列了个官方rfc中的例子

**示例**

在vue的rfc中[reactivity-effect-scope](https://github.com/vuejs/rfcs/blob/master/active-rfcs/0041-reactivity-effect-scope.md)有这样的一个例子，如果有个监控鼠标位置的hook（useMouse），需要监听`mousemove`事件，如果在多个组件调用了这个hook，而内部是通过在`onUnmounted`钩子来移除`mousemove`监听器，`onUnmounted`耦合在每个组件实例，则无法以更有效率的方式共享这个`mousemove`监听器（即）

为了做到在组件间共享`useMouse`的响应式effect和监听器，可以创建一个函数来管理scope如下（这个hook也在vueuse中）

```js
function createSharedComposable(composable) {
  let subscribers = 0 // 订阅计数器，每次调用subscribers + 1
  let state, scope

  const dispose = () => {
    if (scope && --subscribers <= 0) { 
      scope.stop()
      state = scope = null
    }
  }

  return (...args) => {
    subscribers++
    if (!state) { // 只在第一次创建effectScope
      scope = effectScope(true) // true为独立的scope作用域
      state = scope.run(() => composable(...args))
    }
    onScopeDispose(dispose) // 在当前活跃的effect作用域上注册一个回调
    return state
  }
}
```

可以看到dispose函数的含义是，当**没有一个组件在使用**它的时候会注销(dispose)创建的`effectScope`，我们只需要如下操作，即可得到一个**在所有组件共享**的`useMouse`

```js
const useSharedMouse = createSharedComposable(useMouse)
```

更加详细的关于`effectScope`的讨论在[Reactivity's `effectScope` API #212](https://github.com/vuejs/rfcs/pull/212)

本来是打算**完全**按照官网的分类进行解读，但是由于VCA（Vue Composition API）的思想，hook被拆的很碎再组合在一起，并不能单纯靠分类进行解读，所以在讲某个分类的hook时也会带上其他的hooks，接下来是对我觉得 常用 或者 有学习到东西 的hook源码进行解读 

## State

### createGlobalState

> 用来做挂组件公共状态管理

简单的单例模式实现，`effectScope`的参数为true时，其不会被父scope收集和回收，独立存在

```ts
export function createGlobalState<T>(
  stateFactory: () => T,
): CreateGlobalStateReturn<T> {
  let initialized = false
  let state: T
  const scope = effectScope(true)

  return () => {
    if (!initialized) {
      state = scope.run(stateFactory)!
      initialized = true
    }
    return state
  }
}
```

### useLocalStorage

```ts
export function useLocalStorage<T extends(string|number|boolean|object|null)> (
  key: string,
  initialValue: MaybeRef<T>,
  options: StorageOptions<T> = {},
): RemovableRef<any> {
  const { window = defaultWindow } = options
  return useStorage(key, initialValue, window?.localStorage, options)
}
```

可以看到就是用`useStorage`来实现的，`useSessionStorage`也是一样，下面我们来看看`useStorage`

### useStorage

在浏览器默认的Storage之上像[store.js](https://github.com/marcuswestin/store.js)一样对数据进行预处理（序列化），否则如果存一个对象在浏览器的Storage存的会是`xxx.toString()`之后的值。但是`useStorage`比起[store.js](https://github.com/marcuswestin/store.js)更进一步的增加了对`Map`和`Set`类型的数据的处理（**`Map`和`Set`存在默认的Storage和store.js都是空对象**）

利用适配器模式，对每种数据类型定义`read`、`write`

```ts
export const StorageSerializers: Record<'boolean' | 'object' | 'number' | 'any' | 'string' | 'map' | 'set', Serializer<any>> = {
  boolean: {
    read: (v: any) => v === 'true',
    write: (v: any) => String(v),
  },
  object: {
    read: (v: any) => JSON.parse(v),
    write: (v: any) => JSON.stringify(v),
  },
  number: {
    read: (v: any) => Number.parseFloat(v),
    write: (v: any) => String(v),
  },
  any: {
    read: (v: any) => v,
    write: (v: any) => String(v),
  },
  string: {
    read: (v: any) => v,
    write: (v: any) => String(v),
  },
  map: { // map序列化
    read: (v: any) => new Map(JSON.parse(v)),
    write: (v: any) => JSON.stringify(Array.from((v as Map<any, any>).entries())),
  },
  set: { // set序列化
    read: (v: any) => new Set(JSON.parse(v)),
    write: (v: any) => JSON.stringify(Array.from((v as Set<any>).entries())),
  },
}
```

接下来判断用户传入的数据类型，进行选择对应的`read`、`write`方法

```ts
export function guessSerializerType<T extends(string | number | boolean | object | null)>(rawInit: T) {
  return rawInit == null
    ? 'any'
    : rawInit instanceof Set
      ? 'set'
      : rawInit instanceof Map
        ? 'map'
        : rawInit instanceof Date
          ? 'date'
          : typeof rawInit === 'boolean'
            ? 'boolean'
            : typeof rawInit === 'string'
              ? 'string'
              : typeof rawInit === 'object'
                ? 'object'
                : !Number.isNaN(rawInit)
                    ? 'number'
                    : 'any'
}

const type = guessSerializerType<T>(rawInit)
const serializer = options.serializer ?? StorageSerializers[type] // 用户还可以定义自己的序列化方法，但必须具有read和write方法
```

下面是对数据的初始化

```ts
const rawInit: T = unref(initialValue) // 用户传入的初始值
const data = (shallow ? shallowRef : ref)(initialValue) as Ref<T> // 用户可以配置是浅响应式还是深，最后会返回给用户

if (!storage)
    return

try {
    const rawValue = storage.getItem(key)
    if (rawValue == null) { // 如果key对应的值为空，那么是首次
        data.value = rawInit // 初始化值
        if (writeDefaults && rawInit !== null) // 用户可配置（writeDefaults）是否首次存入Storage
            storage.setItem(key, serializer.write(rawInit))
    }
    else {
        data.value = serializer.read(rawValue) // 如果Storage存在对应值，序列化读取
    }
}
catch (e) {
    onError(e) // 用户传入的错误处理函数
}
```

然后是**根据监听data的变化存入Storage**，`watchWithFilter`可以看做是新增了`eventFilter`选项的`watch`，在后文会描述，也是vueuse很重要的一个特性

```ts
watchWithFilter(
    data,
    () => {
        try {
            if (data.value == null)
                storage.removeItem(key)
            else
                storage.setItem(key, serializer.write(data.value))
        }
        catch (e) {
            onError(e)
        }
    },
    {
        flush, // 均是用户可配置的选项
        deep,
        eventFilter,
    },
)
```

是不是觉得很OK了，但仍有巨隐蔽的bug（哈哈哈)，为了支持响应式的存取Storage，用了ref类型的data，**在同源下的多个标签的情况下，其中一个页面对另一个页面用`useStorage`修改了数据，但另一个页面useStorage内部的data还是原来的值**

解决方式：监听[storage](https://developer.mozilla.org/zh-CN/docs/Web/API/StorageEvent)事件（当页面使用的storage被其他页面修改时会触发）

```ts
function read(event?: StorageEvent) { // 将上面初始化值的逻辑封装为read函数
    if (!storage || (event && event.key !== key))
        return

    try {
        const rawValue = event ? event.newValue : storage.getItem(key)
		// 读值省略...
    }
    catch (e) {
        onError(e)
    }
}

if (window && listenToStorageChanges) // 用户也可以配置（listenToStorageChanges）不监听
    useEventListener(window, 'storage', e => setTimeout(() => read(e), 0)) // 重新读取值给data
```

## Browser

> 浏览器相关的hook，基于web暴露的api来实现

### 可配置全局对象

Browser hook的源码开头都有或类似下面这一段

```typescript
export const defaultWindow = /* #__PURE__ */ isClient ? window : undefined

export function useXXX<T>(options: ConfigurableWindow = {}) {
  const { window = defaultWindow } = options
  window.xxx
  // ...
}
```

用户可以配置当前window对象，这种配置方式对于使用**iframe**和**测试环境**不同的window对象十分有用

### useEventListener

先上一个简易版本的`useEventListener`，利用VCA，将事件的监听和注销放在一个函数中处理

```js
function useEventListener (target, listener, options, target = window) {
  onMounted(() => {
    target.addEventListener(type, listener, options)
  })

  onUnmounted(() => {
    target.removeEventListener(type, listener, options)
  })
}
```

除此之外，源码还提供了当`target`的类型是`ref`时的情况，为了减少篇幅，下面的代码删减了参数为空的边界情况

```ts
export function useEventListener(...args: any[]) {
  let target: MaybeRef<EventTarget> | undefined = defaultWindow;
  let event: string
  let listener: any
  let options: any

  [target, event, listener, options] = args

  let cleanup = noop

  const stopWatch = watch(
    () => unref(target),
    (el) => {
      cleanup()
      if (!el)
        return

      el.addEventListener(event, listener, options)

      cleanup = () => {
        el.removeEventListener(event, listener, options)
        cleanup = noop
      }
    },
    { immediate: true, flush: 'post' }, // 为什么flush是post？下面会讲解
  )

  const stop = () => {
    stopWatch()
    cleanup()
  }

  tryOnScopeDispose(stop)

  // 还设计了返回stop函数，可以调用useEventListener返回的函数直接回收
  return stop
}
```

#### 为什么需要支持ref target

用`unref`得到`ref`中的dom元素，监听dom元素的改变以重新监听事件，应用场景就放官网例子

```html
<template>
  <div v-if="cond" ref="element">Div1</div>
  <div v-else ref="element">Div2</div>
</template>
```

```ts
import { useEventListener } from '@vueuse/core'

const element = ref<HTMLDivElement>()
useEventListener(element, 'keydown', (e) => { console.log(e.key) })
```

#### 为什么需要watch设置flush为post

详情请看[#356 Bug: useEventListener doesn't work correctly with v-if](https://github.com/vueuse/vueuse/issues/356)，意思就是[新版vue](https://github.com/vuejs/vue-next/issues/1706#issuecomment-666258948%5C)的`watch`默认在所有组件update前执行，那么ref没有更新导致事件没有更新，所以需要在组件update完毕后更新事件。

#### tryOnScopeDispose是啥

先判断是否有活跃的effectScope，有的话用`onScopeDispose`在其上注册fn回调

```ts
export function tryOnScopeDispose(fn: Fn) {
  if (getCurrentScope()) {
    onScopeDispose(fn)
    return true
  }
  return false
}
```

v10.4.1更新：支持event传递数组，方便浏览器兼容事件

### useEyeDropper

`new window.EyeDropper()`可以获取[**取色器**](https://developer.mozilla.org/zh-CN/docs/Web/API/EyeDropper)的功能

### usePreferredReducedMotion

[prefers-reduced-motion](https://developer.mozilla.org/en-US/docs/Web/CSS/@media/prefers-reduced-motion)用于检测用户的系统是否被开启了动画减弱功能

```ts
useMediaQuery('(prefers-reduced-motion: reduce)', options)
```

### usePreferredContrast

[prefers-contrast](https://developer.mozilla.org/en-US/docs/Web/CSS/@media/prefers-contrast)判断用户喜欢何种对比度（more、less、custom）

### useMediaQuery

`window.matchMedia(query)`返回`MediaQueryList`类型

```ts
interface MediaQueryList extends EventTarget { // 浏览器原生类型
    readonly matches: boolean; // 是否匹配
    readonly media: string;    // 媒体查询值
    onchange: ((this: MediaQueryList, ev: MediaQueryListEvent) => any) | null; // 媒体查询改变回调
	// 事件绑定相关...
}

let mql: MediaQueryList = window.matchMedia('(max-width: 600px)');
```

### usePermission

[navigator.permissions](https://developer.mozilla.org/zh-CN/docs/Web/API/Navigator/permissions)可以获取用户权限（摄像头、麦克风等）

```ts
const isSupported = Boolean(navigator && 'permissions' in navigator)
const state = ref<PermissionState | undefined>() // 存权限状态

const onChange = () => {
    if (permissionStatus)
        state.value = permissionStatus.state
}

try {
    permissionStatus = await navigator!.permissions.query({
        name: 'camera' // https://w3c.github.io/permissions/#enumdef-permissionname
    })
    useEventListener(permissionStatus, 'change', onChange) // 有三个状态 denied" | "granted" | "prompt"
    onChange()
}
catch {
    state.value = 'prompt'
}
```

### usePreferredLanguages

[navigator.languages](https://developer.mozilla.org/en-US/docs/Web/API/Navigator/languages)可以获取用户的偏好语言，可以通过`languagechange`事件监听改变

```ts
const value = ref<readonly string[]>(navigator.languages)

useEventListener(window, 'languagechange', () => {
    value.value = navigator.languages
})
```

### useShare

[navigator.share](https://developer.mozilla.org/zh-CN/docs/Web/API/Navigator/share)可以进行分享

```ts
const isSupported = navigator && 'canShare' in navigator

if (isSupported) {
    granted = navigator.canShare({ // 判断是否能被分享
        title?: string
        files?: File[]
        text?: string
        url?: string
	})

    if (granted)
        return navigator.share!(data)
}
```

### useWakeLock

 [Screen Wake Lock API](https://developer.mozilla.org/en-US/docs/Web/API/Screen_Wake_Lock_API)可以阻止设备变暗或锁屏

```ts
const isSupported = navigator && 'wakeLock' in navigator

function request(type: WakeLockType) {
    if (!isSupported) return
    wakeLock = await navigator.wakeLock.request(type)
}

function release() {
    if (!isSupported || !wakeLock) return
    await wakeLock.release() // 释放WakeLockSentinel
    wakeLock = null
}
```

### useVibrate

 [Vibration API](https://developer.mozilla.org/en-US/docs/Web/API/Vibration_API)可以使设备振动

```ts
window.navigator.vibrate([200, 100, 200]); // 设备振动 200 ms，然后暂停 100 ms，然后再次振动设备 200 ms
navigator.vibrate(0); // 停止振动
```

### useWebNotification

[Notification](https://developer.mozilla.org/en-US/docs/Web/API/notification)可以给用户发送消息

### useFullscreen

[Fullscreen_API](https://developer.mozilla.org/zh-CN/docs/Web/API/Fullscreen_API)用以控制全屏展示，虽然常见，但是看源码对兼容性有了多一层考虑

```typescript
// 请求全屏
const requestMethod = computed<'requestFullscreen' | undefined>(() => {
  return [
    'requestFullscreen',
    'webkitRequestFullscreen',
    'webkitEnterFullscreen',
    'webkitEnterFullScreen',
    'webkitRequestFullScreen',
    'mozRequestFullScreen',
    'msRequestFullscreen',
  ].find(m => (document && m in document) || (targetRef.value && m in targetRef.value)) as any
})

// 退出全屏
const exitMethod = computed<'exitFullscreen' | undefined>(() => {
  return [
    'exitFullscreen',
    'webkitExitFullscreen',
    'webkitExitFullScreen',
    'webkitCancelFullScreen',
    'mozCancelFullScreen',
    'msExitFullscreen',
  ].find(m => (document && m in document) || (targetRef.value && m in targetRef.value)) as any
})

// 当前是否处于全屏状态的判断
const fullscreenEnabled = computed<'fullscreenEnabled' | undefined>(() => {
  return [
    'fullScreen',
    'webkitIsFullScreen',
    'webkitDisplayingFullscreen',
    'mozFullScreen',
    'msFullscreenElement',
  ].find(m => (document && m in document) || (targetRef.value && m in targetRef.value)) as any
})

// 获取当前全屏元素的方法名
const fullscreenElementMethod = [
  'fullscreenElement',
  'webkitFullscreenElement',
  'mozFullScreenElement',
  'msFullscreenElement',
].find(m => (document && m in document)) as 'fullscreenElement' | undefined

```

利用适配器的方式进行调用，`target[requestMethod.value]()`

下面是不同浏览器监听全局变化的事件名

```TS
const eventHandlers = [
  'fullscreenchange',
  'webkitfullscreenchange',
  'webkitendfullscreen',
  'mozfullscreenchange',
  'MSFullscreenChange',
]

// useEventListener支持传数组
useEventListener(() => unrefElement(targetRef), eventHandlers, handlerCallback, false)
```

## Sensors

### onStartTyping

在开始输入的时候执行，我们可以拿来做到**不管编辑器是否聚焦，只要输入就聚焦到编辑器并输入**

```ts
export function onStartTyping(callback: (event: KeyboardEvent) => void, options: ConfigurableDocument = {}) {
  const { document = defaultDocument } = options

  const keydown = (event: KeyboardEvent) => {
      isTypedCharValid(event) // 检测是否是有效输入 0...9 A...Z a...z
      && callback(event)
  }

  if (document)
    useEventListener(document, 'keydown', keydown, { passive: true })
}
```

在上面的基础上还需要做一点，如果已经聚焦到`non-editable elements`，那么就不执行，如何检测`non-editable elements`呢，如下

```ts
function isFocusedElementEditable() {
  const { activeElement, body } = document

  if (!activeElement)
    return false

  if (activeElement === body)
    return false

  switch (activeElement.tagName) {
    case 'INPUT':
    case 'TEXTAREA':
      return true
  }
    
  return activeElement.hasAttribute('contenteditable')
}
```

### useMagicKeys

恰如其名，**magic**，先上用法

```ts
const { space, shift } = useMagicKeys()

watch(space, (v) => {
  if (v)
    console.log('space has been pressed')
})
```

又没传参，他怎么监听到我要的`space`和`shift`，难不成他返回了所有键？那当然不是，上源码（抽离了关键逻辑）

```ts
export function useMagicKeys(options: UseMagicKeysOptions<boolean> = {}): any {
  const current = reactive(new Set<string>())
  const obj = {
    toJSON() { return {} },
    current,
  }
  const refs: Record<string, any> = useReactive ? reactive(obj) : obj
  
  useEventListener(target, 'keydown', (e: KeyboardEvent) => {
    updateRefs(e, true) // 更新按下的键到refs，键为e.key?.toLowerCase()，值为true
  }, { passive })
    
  useEventListener(target, 'keyup', (e: KeyboardEvent) => {
    updateRefs(e, false) // 更新按下的键到refs，键为e.key?.toLowerCase()，值为false
  }, { passive })
  
  const proxy = new Proxy(
    refs,
    {
      get(target, prop, rec) {
          
        if (!(prop in refs)) {
          // 初始将对应的键设为false，如'shift'
          refs[prop] = ref(false)
        }
        const r = Reflect.get(target, prop, rec)
        return useReactive ? toValue(r) : r
      },
    },
  )

  return proxy as any
}

```

这样refs只存了取的键的状态，而不是所有键的状态

### usePageLeave

用[MouseEvent.relatedTarget](https://developer.mozilla.org/zh-CN/docs/Web/API/MouseEvent/relatedTarget)检测是否离开屏幕

```ts
export function usePageLeave(options: ConfigurableWindow = {}) {
  const handler = (event: MouseEvent) => {
    const from = event.relatedTarget || event.toElement // 如果为null，则移出屏幕
    isLeft.value = !from
  }

  if (window) {
    useEventListener(window, 'mouseout', handler, { passive: true })
    useEventListener(window.document, 'mouseleave', handler, { passive: true })
    useEventListener(window.document, 'mouseenter', handler, { passive: true })
  }

  return isLeft
}

```

### useSpeechSynthesis

利用[SpeechSynthesis](https://developer.mozilla.org/en-US/docs/Web/API/SpeechSynthesis)来做到语音阅读

```ts
export function useSpeechSynthesis(text: MaybeRefOrGetter<string>, options: UseSpeechSynthesisOptions = {}) {
  // ...
  const synth = window && (window as any).speechSynthesis as SpeechSynthesis;
    
  const bindEventsForUtterance = (utterance: SpeechSynthesisUtterance) => {
    utterance.lang = toValue(lang)
    utterance.voice = toValue(options.voice) || null
    utterance.pitch = toValue(pitch)
    utterance.rate = toValue(rate)
    utterance.volume = volume

    utterance.onstart = () => {
      isPlaying.value = true
      status.value = 'play'
    }

    utterance.onpause = () => {
      isPlaying.value = false
      status.value = 'pause'
    }

    utterance.onresume = () => {
      isPlaying.value = true
      status.value = 'play'
    }

    utterance.onend = () => {
      isPlaying.value = false
      status.value = 'end'
    }

    utterance.onerror = (event) => {
      error.value = event
    }
  }

  const utterance = computed(() => {
    isPlaying.value = false
    status.value = 'init'
    const newUtterance = new SpeechSynthesisUtterance(spokenText.value)
    bindEventsForUtterance(newUtterance)
    return newUtterance
  })

  const speak = () => {
    synth!.cancel()
    utterance && synth!.speak(utterance.value)
  }

  const stop = () => {
    synth!.cancel()
    isPlaying.value = false
  }

  tryOnScopeDispose(() => {
    isPlaying.value = false
  })

  // ...
}
```

## Animation

### useTransition

**可配置项**

```ts
type CubicBezierPoints = [number, number, number, number]
type EasingFunction = (n: number) => number

export function useTransition(
  source: Ref<number | number[]> | MaybeRef<number>[],
  options: TransitionOptions = {},
): ComputedRef<any> {
  const { // 时间相关单位均为ms
    delay = 0,           // delay 后执行
    disabled = false,    // 是否关闭
    duration = 1000,     // 持续时间
    onFinished = noop,   // 完成后执行
    onStarted = noop,    // 开始时执行
    transition = linear, // 转化算法 MaybeRef<EasingFunction | CubicBezierPoints>，默认值是 x => x
  } = options

  // ...
}
```

**预处理**

```ts
export function useTransition(
  source: Ref<number | number[]> | MaybeRef<number>[],
  options: TransitionOptions = {},
): ComputedRef<any> {
  // 可配置项
  
  // 创建当前的过渡函数，如果用户自定义了过渡函数则不做处理，否则根据贝赛尔曲线值创建过渡函数
  const currentTransition = computed(() => {
    const t = unref(transition)
    return isFunction(t) ? t : createEasingFunction(t)
  })

  // 用unref处理source参数，如果source是数组则依次unref
  const sourceValue = computed(() => {
    const s = unref<number | MaybeRef<number>[]>(source)
    return isNumber(s) ? s : s.map(unref) as number[]
  })
  
  // 规格化source，整成数组
  const sourceVector = computed(() => isNumber(sourceValue.value) ? [sourceValue.value] : sourceValue.value)

  // ...
}

// 创建过渡函数，纯数学
function createEasingFunction([p0, p1, p2, p3]: CubicBezierPoints): EasingFunction {
  const a = (a1: number, a2: number) => 1 - 3 * a2 + 3 * a1
  const b = (a1: number, a2: number) => 3 * a2 - 6 * a1
  const c = (a1: number) => 3 * a1

  const calcBezier = (t: number, a1: number, a2: number) => ((a(a1, a2) * t + b(a1, a2)) * t + c(a1)) * t

  const getSlope = (t: number, a1: number, a2: number) => 3 * a(a1, a2) * t * t + 2 * b(a1, a2) * t + c(a1)

  const getTforX = (x: number) => {
    let aGuessT = x

    for (let i = 0; i < 4; ++i) {
      const currentSlope = getSlope(aGuessT, p0, p2)
      if (currentSlope === 0)
        return aGuessT
      const currentX = calcBezier(aGuessT, p0, p2) - x
      aGuessT -= currentX / currentSlope
    }

    return aGuessT
  }

  return (x: number) => p0 === p1 && p2 === p3 ? x : calcBezier(getTforX(x), p1, p3)
}
```

提供了几种过渡预设

```ts
export const TransitionPresets: Record<string, CubicBezierPoints | EasingFunction> = {
  easeInSine: [0.12, 0, 0.39, 0],
  easeOutSine: [0.61, 1, 0.88, 1],
  // ...
}
```

详情可以看 [缓动函数](https://easings.net/cn#)、 [MDN：缓动函数](https://developer.mozilla.org/en-US/docs/Web/CSS/easing-function#easing_functions)

**关键逻辑**

```ts
const outputVector = ref(sourceVector.value.slice(0)) // 用以存储输出

// 过渡主流程，可以暂停
const { resume, pause } = useRafFn(() => { // 可以暂停、恢复的 requestAnimationFrame
  const now = Date.now()
  const progress = clamp(1 - ((endAt - now) / currentDuration), 0, 1) // 过渡进度，clamp将值限制在0-1之间

  outputVector.value = startVector.map((val, i) => val + ((diffVector[i] ?? 0) * currentTransition.value(progress))) // 更新output

  if (progress >= 1) { // 完成过渡
    pause() // 暂停 requestAnimationFrame
    onFinished() // 执行options中的完成回调
  }
}, { immediate: false })

const timeout = useTimeoutFn(start, delay, { immediate: false }) // delay毫秒后执行start函数

const start = () => {
  pause() // 暂停 过渡

  currentDuration = unref(duration)
  diffVector = outputVector.value.map((n, i) => (sourceVector.value[i] ?? 0) - (outputVector.value[i] ?? 0)) // 初始值和当前值的diff
  startVector = outputVector.value.slice(0) // 拷贝初始值
  startAt = Date.now() // 开始时间
  endAt = startAt + currentDuration // 结束时间，用在计算progress

  resume() // 开始过渡
  onStarted() // 执行options中的开始回调
}

watch(sourceVector, () => { // 监听sourceVector的变化
  if (unref(disabled)) { // 如果暂停过渡，将输出还原回初始值
    outputVector.value = sourceVector.value.slice(0)
  }
  else {
    if (unref(delay) <= 0) start() // 如果没有延迟，则开始过渡
    else timeout.start() // 如果有延迟，则开始上面创建好的timeout
  }
}, { deep: true })

return computed(() => {
  const targetVector = unref(disabled) ? sourceVector : outputVector // 如果暂停过渡，将输出还原回初始值
  return isNumber(sourceValue.value) ? targetVector.value[0] : targetVector.value // 为值解开数组包装，数组原样返回
})
```

## @Sound

可以简单的让你的网站带上声音

```ts
import buttonSfx from '../assets/sounds/button.mp3'

const { play } = useSound(buttonSfx)
```

demo：https://sound.vueuse.org/

