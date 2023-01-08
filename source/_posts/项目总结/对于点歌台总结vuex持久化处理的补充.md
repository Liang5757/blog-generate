---
title: 对于点歌台总结vuex持久化处理的补充
date: 2020-09-16 20:25:18
tags:
  - vue
categories:
  - 前端
---
> 针对刷新时vuex数据丢失，使用了vuex-persistedstate对vuex进行持久化处理
> 官方github：https://github.com/robinvdvleuten/vuex-persistedstate

本文主要是对官网不够详细的案例进行补充，官网只讲了对vuex完全存储和对vuex模块的完全存储。
但是我想对某一模块内部一些变量进行存储，谷歌了好多都没找到写法，自己试出来了。
<!--more-->
1. 首先引入vuex-persistedstate

```js
import createPersistedState from "vuex-persistedstate";
```

2. 利用官网给的reducer减少持久化的数据

![image-20200916201819305](image-20200916201819305.png)

```ts
const store = new Vuex.Store({
    state,
    getters,
    mutations,
    actions,
    modules: {
        projectDetail
    },
    // @ts-ignore
    plugins: [createPersistedState({
        reducer(val) {
            return {
                staffId: val.staffId,
                projectDetail: {
                    curPjId: val.projectDetail.curPjId,
                }
            }
        }
    })]
})
```

可以看到该vuex结构具有projectDetail模块，我想对该模块内的curPjId进行**单独**存储，在reducer放回的对象的键设为模块名，里面写着想要持久化的变量，就可以了。

会在localstorage存储为如下所示

![image-20200916202437681](image-20200916202437681.png)