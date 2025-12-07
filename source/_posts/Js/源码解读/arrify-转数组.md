---
title: 【源码解读】arrify 转数组
date: 2022-05-21 11:33:05
tags:
  - js
categories:
  - 源码解读
---
半年没写博客，从简单的源码开始启动
<!--more-->

> github: https://github.com/sindresorhus/arrify

### 功能代码

比较简单，就先展示一下全部js代码

```js
export default function arrify(value) {
	if (value === null || value === undefined) {
		return [];
	}

	if (Array.isArray(value)) {
		return value;
	}

	if (typeof value === 'string') {
		return [value];
	}

	if (typeof value[Symbol.iterator] === 'function') {
		return [...value];
	}

	return [value];
}
```

在我看来和Array.from的区别如下

|                             | Array.from                                                   | arrify                        |
| --------------------------- | ------------------------------------------------------------ | ----------------------------- |
| undefined                   | 报错：Uncaught TypeError: undefined is not iterable (cannot read property Symbol(Symbol.iterator)) | []                            |
| Null                        | 报错：Uncaught TypeError: undefined is not iterable (cannot read property Symbol(Symbol.iterator)) | []                            |
| document.querySelector('*') | 带所有元素的数组                                             | [document.querySelector('*')] |

作者对于array like处理的[回应](https://github.com/sindresorhus/arrify/issues/2)，翻译一下：故意这么整的，arrify应当只是数组化，而不是对类数组进行转换，对类数组转换可以用`Array.from`

ps：在我尝试的时候，用chrome控制台Array.from(document.querySelector('*'))，会得到一个**空数组**，但是本地创建的html测试是没问题的，不知道为什么

#### 关于string的处理

为什么不直接去掉value === 'string'这个if语句块，这样在最后也能返回[value]？

答：string有Symbol.iterator属性，所以会执行[...value]

### 类型定义

#### 重载定义？

最开始的类型定义是用**重载**的方式，对不同参数类型进行处理，代码如下

```ts
declare function arrify(value: null | undefined): [];
declare function arrify(value: string): [string];
declare function arrify<ValueType>(value: ValueType[]): ValueType[];
declare function arrify<ValueType>(value: ReadonlyArray<ValueType>): ReadonlyArray<ValueType>;
declare function arrify<ValueType>(value: Iterable<ValueType>): ValueType[];
declare function arrify<ValueType>(value: ValueType): [ValueType];
```

但是在该[issus](https://github.com/sindresorhus/arrify/issues/8)提出了，参数不支持联合类型，case如下

```ts
arrify(Boolean() ? [1, 2] : 3);
```

参数会被ts推断为`number | number[]`，并不满足上面所有重载方法的定义，考虑联合类型后，各种类型组合的数量太多了，函数重载的定义方式无法满足需求。

#### 改进版

类型定义如下，其实也和js的判断一一对应

```typescript
export default function arrify<ValueType>(
   value: ValueType
): ValueType extends (null | undefined)
   ? []
   : ValueType extends string
      ? [string]
      : ValueType extends readonly unknown[]
         ? ValueType
         : ValueType extends Iterable<infer T>
            ? T[]
            : [ValueType];
```

使用extend后，会将联合类型的每一个取出来执行判断，得出结果后再合并成联合类型，例子：

```typescript
type value<ValueType> = ValueType extends (null | undefined)
	? []
	: ValueType extends string
	? [string]
	: ValueType extends readonly unknown[]
	? ValueType
	: ValueType extends Iterable<infer T>
	? T[]
	: [ValueType];

type test = value<number | string[]> // [number] | string[]
```

### 总结

还是比较老的仓库，他的一些设计理念我不是很认同，我觉得给人用的第三方库就应该能够处理所有的情况，而不是为了设计理念来严格定义，不对一些情况进行处理

