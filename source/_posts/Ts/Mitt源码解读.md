---
title: Mitt源码解读
date: 2021-12-15 19:45:48
tags:
  - ts
categories:
  - 前端
  - 源码解读
---

github地址：https://github.com/developit/mitt

> 200b大小的event bus库

<!--more-->

## 类型系统

让人眼前一亮的是，该库对类型有着较强规范，用户可以很方便的用下面的方式定义事件类型，在该[issus](https://github.com/developit/mitt/issues/106)被提出

```typescript
type Events = {
  foo: string; // 事件名: 参数类型
  bar?: number;
};

const emitter = mitt<Events>(); // inferred as Emitter<Events>

emitter.on('foo', (e) => {}); // 'e' has inferred type 'string'

emitter.emit('foo', 42); // Error: Argument of type 'number' is not assignable to parameter of type 'string'. (2345)
```

那它是如何做到的呢，先把实现代码展示出来再分部讲解（直接看懂的可以跳过讲解）。

```typescript
export type EventType = string | symbol;
export type Handler<T = unknown> = (event: T) => void;

export default function mitt<Events extends Record<EventType, unknown>>(
	all?: EventHandlerMap<Events> // 可以暂时忽略这里
): Emitter<Events> {
    type GenericEventHandler =
    | Handler<Events[keyof Events]>
    | WildcardHandler<Events>; // 为了适配用 *

	return {
        // ...
		on<Key extends keyof Events> (type: Key, handler: GenericEventHandler) {
			// ...
		},
		off<Key extends keyof Events> (type: Key, handler?: GenericEventHandler) {
			// ... 
		},
        emit<Key extends keyof Events>(type: Key, evt?: Events[Key])  {
            // ...
        }
	}
}
```

### Record

`Record<K, T>`是typescript内置的类型函数

```ts
// Construct a type with a set of properties K of type T
type Record<K extends keyof any, T> = {
    [P in K]: T;
};
```

代表的意思是从第一个参数获取所有键，值为第二个参数，源码中`Record<EventType, unknown>`即为

```typescript
export type EventType = string | symbol;
Record<EventType, unknown> === {
    [P in string | symbol]: unknown;
}
```

`Events extends Record<EventType, unknown>`利用泛型约束将用户传入的Events类型约束为上述类型

### 类型编程

就像余华老师睡一觉醒来后有了这么一个题目叫《活着》，觉得题目非常好就开始写了，第一次见到[Linbudu的博客](https://linbudu.top/ts-type-programming)，见到了一个词——**类型编程**，有了这么一个概念（虽然不知道是在何时何地被提出的），突然很多知识就有了一个系统的名称，引导人们探索这个系统下的各种应用，下面就是一个很简单的一个例子。

```ts
export type Handler<T = unknown> = (event: T) => void;

type GenericEventHandler = Handler<Events[keyof Events]>
```

`Events[keyof Events]`就是将用户定义的Events类型的值类型全部取出，在最上面的使用方式的例子里，会有如下的转化结果。

```ts
type Events = {
  foo: string; // 事件名: 参数类型
  bar?: number;
};

Events[keyof Events] === string | number;
type GenericEventHandler = Handler<Events[keyof Events]> === (event: string | number) => void
```

再看`on`、`off`、`emit`函数定义，去除了 支持type为`*`的功能

```ts
on<Key extends keyof Events> (type: Key, handler: Handler<Events[Key]>) {
    // ...
}
off<Key extends keyof Events> (type: Key, handler?: Handler<Events[Key]>) {
    // ...
}
emit<Key extends keyof Events>(type: Key, evt?: Events[Key])  {
    // ...
}
```

因此我们在`on`、`off`和`emit`只能注册或注销**事件名**和**handler的参数**与用户定义Events类型一致的事件

## 代码逻辑

### 可替换的all

all用来存储用户注册的事件，mitt会初始化一个Map作为默认的，但是用户可以传入自己的Map。

```ts
export default function mitt<Events extends Record<EventType, unknown>>(
	all?: EventHandlerMap<Events>
): Emitter<Events> {
    all = all || new Map();
}
```

### on

```ts
on<Key extends keyof Events>(type: Key, handler: GenericEventHandler) {
    const handlers: Array<GenericEventHandler> | undefined = all!.get(type);
    if (handlers) {
        handlers.push(handler);
    }
    else {
        all!.set(type, [handler] as EventHandlerList<Events[keyof Events]>);
    }
}
```

如果已经注册同名事件则直接将handler推进数组，否则创建个新数组存放hanlder。

### off

```ts
off<Key extends keyof Events>(type: Key, handler?: GenericEventHandler) {
    const handlers: Array<GenericEventHandler> | undefined = all!.get(type); // 对应事件名的handler数组
    if (handlers) {
        if (handler) {
            handlers.splice(handlers.indexOf(handler) >>> 0, 1);
        }
        else {
            all!.set(type, []);
        }
    }
}
```

有一个秀操作的点，当用户传入handler要**删除对应事件名下的对应handler**时，有这么一段。

```ts
handlers.splice(handlers.indexOf(handler) >>> 0, 1);
```

我们知道

- `indexOf`没有在数组中找到对应的值，那么就会返回-1
- `>>> 0`（ 零填充右位移0位）不是没变？
- 而`splice(-1, 1)`表示删除最后一位

综合起来看，如果`-1 >>> 0`没变那么就会误删最后一个handler。

让我们看看`>>> 0`是什么魔法

```ts
console.log(-1 >>> 0); // 4294967295
```

js number采用的是IEEE 754双精度浮点，**\>>>会把number截断成32位int进行位移**，更具体的不展开。如果没找到对应的handler，代码即为`handlers.splice(4294967295, 1)`，我们当然不可能存如此多的handler，也就不可能删除4294967295下标对应的handler，比起我们判断`indexOf`是否为-1缩短了代码（什么叫**Microscopic**啊？）

### emit

- 适配type为\*的情况，为所有事件都注册handler，emit时取出\*中的所有handler执行

```ts
emit<Key extends keyof Events>(type: Key, evt?: Events[Key]) {
    let handlers = all!.get(type);
    if (handlers) {
        (handlers as EventHandlerList<Events[keyof Events]>)
            .slice()
            .map((handler) => {
            handler(evt!);
        });
    }

    handlers = all!.get('*');
    if (handlers) {
        (handlers as WildCardEventHandlerList<Events>)
            .slice()
            .map((handler) => {
            handler(type, evt!);
        });
    }
}
```

## 题外话

### Once

虽然源码并没有实现，但是既然是事件总线的文章，还是提一提。在我之前面试的时候经常被问到事件总线是如何实现的，其中就有Once的实现，我没有深入思考，只想着实现就完事了：**给所有handler用对象包裹一层，然后加一个Once字段标识，在emit时判断Once是否为true，true的话执行完注销事件**，但是这样`on`和`emit`都需要修改，多创建了一个对象并且不优雅。

我们可以充分利用js的灵活性，用一个函数包裹用户的handler，在该函数中off掉handler。

```ts
once(type: Key, handler: GenericEventHandler) {
    function on () {
        const self: Emitter<Events> = this;
        function listener () {
            self.off(type, listener);
            handler.apply(this, arguments);
        }

        this.on(type, listener);
    }
}
```

但是上面的代码有一个很隐蔽的问题：**由于是用一个新的函数去包裹，在off时传旧handler去注销once事件无法成功**

**解决方式**

在listener函数中存旧的handler

```ts
once(type: Key, handler: GenericEventHandler) {
    function on () {
        const self: Emitter<Events> = this;
        function listener () {
            self.off(type, listener);
            handler.apply(this, arguments);
        }
        listener.fn = handler; // 增加了这里
        
        this.on(type, listener);
    }
}
```

不过用这种方式的话，我们的`>>>`魔法就不能使用的（这是mitt没有once的原因？）

```typescript
off<Key extends keyof Events>(type: Key, handler?: GenericEventHandler) {
    const handlers: Array<GenericEventHandler> | undefined = all!.get(type);
    if (handlers) {
        if (handler) {
            for (let i = 0; i < handlers.length; i++) {
                if (handler === handlers[i] || handler === handlers[i].fn)
                    handlers.splice(i, 1);
            }
        }
        else {
            all!.set(type, []);
        }
    }
}
```

### unknown

在该[Stronger typing #114](https://github.com/developit/mitt/pull/114)commit中，大量的any被替换为unknown

关于unknown的特点可看[[译] TypeScript 3.0: unknown 类型](https://juejin.cn/post/6844903866073350151)

