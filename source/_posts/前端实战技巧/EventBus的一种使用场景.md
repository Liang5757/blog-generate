---
title: EventBus的一种使用场景
date: 2020-09-17 19:38:58
tags:
  - vue
  - 技巧
categories:
  - 前端
---
> 所谓事件总线，就是实例化Vue对象，在该实例上通过`$on`绑定事件、`$emit`触发事件、`$off`解绑事件，进行组件通信。

## 一、使用

实例化Vue对象，并挂载到Vue.prototype

```js
Vue.prototype.$bus = new Vue();
```

## 二、场景

> 在多个页面复用一个组件时，每个页面需要有点击按钮后，触发不同的事件

当然可以监听路由进行判断调用不同的函数，但是这样会在一个组件内写上很多其他组件应该触发的事件。

**我们利用EventBus可以做到组件间的解耦**

## 三、使用方法

1.我们在各个页面上写好事件触发后调用的函数，然后在mounted（如果用keep-alive则是在activated，否则只挂载一次）`this.$bus.$on`上绑定该事件，在beforeDestory（如果用keep-alive则是在deactived，否则无法解绑事件）上用`this.$bus.$off`解绑事件。

```js
mounted() {
    this.$bus.$on("save", this.save);
}

beforeDestory() {
    this.$bus.$off("save", this.save);
}

save() {
    console.log("我没有写在复用组件上哦！")
}
```

2.然后在复用的组件上触发事件

```
this.$bus.$emit('save');
```

也算是第一次尝试使用EventBus把，很好的降低了组件间的耦合度