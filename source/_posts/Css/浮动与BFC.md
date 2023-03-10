---
title: 浮动与BFC
date: 2020-07-31 17:11:21
tags:
  - CSS
categories:
  - 前端
---

### 1.  浮动的工作原理

浮动会**脱离**正常的文档流，并吸附到其父容器左边，正常布局中位于浮动元素下的内容会**围绕**着浮动元素

### 2.可选值

| 值    | 效果                                   |
| ----- | -------------------------------------- |
| none  | 默认值，元素默认在文档流中排列         |
| left  | 元素会立即脱离文档流，向页面的左侧浮动 |
| right | 元素会立即脱离文档流，向页面的右侧浮动 |

<!--more-->

### 3.浮动的规则

**内联元素变块元素**
当为一个元素设置浮动以后（float属性是一个非none的值），元素会立即脱离文档流，元素脱离文档流以后，它下边的元素会立即向上移动，元素浮动以后，会尽量向页面的左上或这是右上漂浮，**直到遇到父元素的边框或者其他的浮动元素**如果浮动元素上边是一个没有浮动的块元素，则**浮动元素不会超过块元素，浮动的元素不会超过他上边的兄弟元素**，**最多最多一边齐**。

### 4.浮动的包裹性

指的是元素尺寸刚好容纳内容, 表现得就像`diaplay:inline-block`

一样具有包裹性的其他属性:

```css
display:inline-block
position:absolute/fixed/sticky
overflow:hidden/scroll
```

### 5. 浮动的破坏性

会使父元素**高度塌陷**——为了实现文字环绕效果

具有破坏性的其他属性:

```css
display:none
position:absolute/fixed/sticky
```

### 6.清除浮动  clear

清除掉其他元素浮动对当前元素产生的影响

**可选值**

| 值    | 效果                             |
| ----- | -------------------------------- |
| none  | 默认值，不清除浮动               |
| left  | 清除左侧浮动元素对当前元素的影响 |
| right | 清除右侧浮动元素对当前元素的影响 |
| both  | 清除两侧浮动元素对当前元素的影响 |

### 7.解决高度塌陷

#### 7.1 BFC

根据W3C的标准，在页面中元素都一个隐含的属性叫做**块级格式化上下文(Block Formatting Context)**，**是块盒子的布局过程发生的区域，也是浮动元素与其他元素交互的区域**，可以设置打开或者关闭，默认是为关闭的。

##### BFC打开方式

1. 设置父元素元素浮动
	\- 使用这种方式开启，虽然可以撑开父元素，但是会导致父元素的宽度丢失。
	\- 而且使用这种方式也会导致下边的元素上移，不能解决问题。
2. 设置元素绝对定位
3. 设置元素为flex-box，grid 或 display: table-cell / table-caption / inline-block
	\- 可以解决问题，但是会导致宽度丢失，不推荐使用这种方式
4. 将元素的overflow设置为一个非visible的值
	\- 此为副作用最小的方式

**BFC特性**

1. 创建了BFC的元素中，**子浮动元素也会参与高度计算**，即父元素的垂直外边距不会和子元素重叠。
2. 开启BFC的元素是一个独立的容器，子元素不会影响外面的元素，反之亦然——可以解决外边距合并的问题
3. 开启BFC的元素
4. 开启BFC的元素可以包含浮动的子元素

由此得到解决高度塌陷的第一种方法

```css
overflow: hidden;
```

**IE6及以下的浏览器中并不支持BFC，所以使用这种方式不能兼容IE6。**

**在IE6中虽然没有BFC，但是具有另一个隐含的属性叫做hasLayout，该属性的作用和BFC类似，所在IE6浏览器可以通过开hasLayout来解决该问题**
**开启方式很多，我们直接使用一种副作用最小的：zoom: 1;**

最后得到以下形式

```css
overflow: hidden;
/* zoom表示放大的意思，只在IE中支持 */
zoom: 1;
```

#### 7.2 用空白元素设置clear:both

可以直接在高度塌陷的**父元素的最后**，**添加一个空白的div**，由于这个div并没有浮动，所以他是可以撑开父元素的高度的，然后在对其进行清除浮动，这样可以通过这个空白的div来撑开父元素的高度，基本没有副作用。

```css
.clear {
	clear: both;
}
```

```html
<div class="box1">
	<div class="box"></div>
	<div class="clear"></div>
</div>
```

虽然可以解决问题，但会在文档中添加多余的结构，不符合结构与表现分离的思想。

#### 7.3 隐式元素

由7.2的方法可以想到用css实现的方法

```css
.clearfix:after {
    /* 添加一个内容 */
    content: "";
    /* 转换为一个块元素 */
    display: block;
    /* 清除两侧的浮动 */
    clear: both;
}
/* 兼容IE6 */
.clearfix {
    zoom:1;
}
```

###### **参考文章**

http://ife.baidu.com/note/detail/id/959
