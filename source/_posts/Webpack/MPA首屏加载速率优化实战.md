---
title: MPA首屏加载速率优化实战
date: 2021-02-14 15:45:27
tags:
  - Webpack
categories:
  - 前端
---
## 背景

​	学校实验室的项目，因为学校只开放给我们一个端口，所以只能把后台管理和学生端合并成多页应用，我是做后台管理的，老师要求某个功能要加上代码高亮，在全局引入highlight.js后发现首屏加载速率不行了，记录一下发现更多问题并优化的过程。
<!--more-->
## 有用的优化

### 1. HighLight.js被放在首屏加载了

首先用`webpack-bundle-analyer`进行构建分析

![image-20210126203158808](image-20210126203158808-1613288799701.png)

发现了两个巨大的包`chunk-vendors.44cd6d2c.js`有2.4MB，`chunk-94715762.bb0c42f0.js`有1.4MB，不幸的是c端需要在首屏同时下载两个大包（此处有误，后面讲解，正确的是左边这个包加入口）才开始渲染，而c端并没有用到`highlight.js`但是他也得等待下载。

然后把`highlight.js`放到封装的组件里引用，然后打包分析

![image-20210126204428302](image-20210126204428302.png)

发现就分离了，**2.4MB的包变成了1.5MB**

### 2. 两个应用使用的element组件没有分离

但是又发现了新的问题——`element-ui`两个应用使用的组件被打包到了一起，即时c端没有使用到诸如`el-upload`、`el-pagination`等组件，但是也要首屏也要下载，想了想`highlight.js`被放到b端入口文件引入就被打进这个包里。

是不是**两个入口使用了同一个element按需引入文件的原因**，然后我给**两个应用各开了自己的按需引入文件**。**打包分析没啥变化**。

问了大哥，大哥甩手就是一个连接https://www.cnblogs.com/HYZhou2018/p/10419703.html

大概就是`vue-cli3`的脚手架配置自动分包的时候是针对单页应用的，下面是`vue-cli3`的配置项

![image-20210126205853873](image-20210126205853873.png)

`splitChunks`默认`minChunks`是1，但是我们是多页应用啊，所以两个应用使用的第三方库全被抽离到一个`chunk-vendor.js`了。

```js
config.optimization.splitChunks({
    cacheGroups: {
        vendors: {
            name: 'chunk-vendors',
            minChunks: 2, // 设置为2，两个应用同时使用才抽离
            test: /node_modules/,
            priority: -10,
            chunks: 'initial'
        },
        common: {}
    }
});
```

再次打包分析

![image-20210126210625547](image-20210126210625547.png)

不仅分离了一些组件，还把一些使用到的第三方库给分离了，这个2.4MB的打包到此为止就变成了**1.2MB**的包。

## 2021/9/2 更新

在上图可以看到，一些node_modules里的第三方库被打包到业务代码中了，这就导致了我们就算只是更新业务代码，但是用户也需要重新下载当前组件引用到的第三方库，我们可以设置更细粒度的分包，在cacheGroups中添加如下配置

```js
teacher: {
  name: "chunk-teacher",
  test: /node_modules/,
  minChunks: 1,
  chunks(chunk) {
    return chunk.name === "teacher";
  },
  priority: -1,
},
student: {
  name: "chunk-student",
  test: /node_modules/,
  minChunks: 1,
  chunks(chunk) {
    return chunk.name === "student";
  },
  priority: -1,
},
```

![image-20210902014110713](image-20210902014110713.png)

可以看到红框圈起来的就是新分出来的包，里面只有被引次数为1的第三方库，保证了业务和依赖的抽离

## 走过的坑

### 1.怎么coding包还是在首屏下载了

已经使用了路由懒加载，为什么coding包还是在首屏下载了，我曾一度以为是没有[`syntax-dynamic-import`](https://babeljs.io/docs/plugins/syntax-dynamic-import/)这个插件的原因，还装过了试了下，但是并没有什么用，而且webpack已经使用动态import来做到懒加载了。

查阅[文章](https://blog.csdn.net/sinat_35538827/article/details/87969834)发现

原来 vue-cli3 默认会把所有通过`import()`按需加载的javascript文件加上 prefetch 。

**prefetch是什么？**在打包后的文件中，查看index.html我们会发现类似这个 <link href=/js/chunk-118075e7.5725ab1a.js rel=prefetch>。<link rel="prefetch">会在页面加载完成后，利用空闲时间提前加载获取用户未来可能会访问的内容。

**prefetch链接会消耗宽带，如果是在移动端，而且存在大量的chunk，那么可以关掉 prefetch 链接，手动选择要提前获取的代码区块。**

```js
//手动选定要提前获取的代码
import(webpackPrefetch: true, './someAsyncComponent.vue')
```

**关闭prefetch:** (官网示例)

```js
// vue.config.js
module.exports = {
  chainWebpack: config => {
    // 移除 prefetch 插件
    config.plugins.delete('prefetch')
 
    // 或者
    // 修改它的选项：
    config.plugin('prefetch').tap(options => {
      options[0].fileBlacklist = options[0].fileBlacklist || []
      options[0].fileBlacklist.push(/myasyncRoute(.)+?\.js$/)
      return options
    })
  }
}
```

### 2. 第三方库怎么这么多重复的bn.js

![image-20210126212734717](image-20210126212734717.png)

可以看到有8个重复的`bn.js`，一个40KB，gzip后10KB，离谱。

但是大小是不一样的，可能**用的版本不同**，目前没有好的方法抽离。。

其实可以用cdn来搞，但是第三方的cdn不稳定，就没搞。

### 3. 打包后mini-css-extract-plugin警告Conflicting order

对应的issus：https://github.com/webpack-contrib/mini-css-extract-plugin/issues/250

是由于组件使用顺序不一致导致的。