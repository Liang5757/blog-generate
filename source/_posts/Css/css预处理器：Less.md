---
title: css预处理器：Less
date: 2020-10-30 20:30:48
tags:
  - Less
categories:
  - 前端
---

### 1.注释

- 以 // 开头的注释，不会被编译到css文件中
- 以 /**/ 包裹的注释会被编译到css文件中

<!--more-->

### 2.变量

#### 2.1普通变量

less中使用@定义一个变量，再以@开头调用

```less
@zero: 0;
* {
    margin: @zero;
    padding: @zero;
}
```

#### 2.2变量作为选择器、属性名或者url

使用@{selector/property/url}调用

```less
@selector: wrap;
@url: "../img/1.jpg";
@w: width;

@{selector}{
    @{w}: 100px;
	background: url("@{url}");
}
```

#### 2.3变量的延迟加载

- less中的加载是有延迟的
- 它会在当前作用域样式未加载之前先加载变量，而且是由内而外，先寻找作用域内的变量，如果没有再寻找作用域外的变量
- less机制是先加载完声明变量再赋值到样式中去

```less
@var: 0;
.class {
@var: 1;
    .brass {
      @var: 2;
      three: @var;  //3
      @var: 3;
    }
  one: @var;  //1
}
```

### 3.嵌套规则

```less
#list{
    list-style: none;
    a{
        float: left;
        /*&代表父级*/
        &:hover{color: red;}
    }
    span{float: right;}
}
```

&的使用（&代表父级）
如果想写串联选择器，而不是写后代选择器，就可以用到&了.
这点对伪类尤其有用如 :hover 和 :focus

### 4.混合

 混合就是一种将一系列属性从一个规则集引入（“混合”）到另一个规则集的方式。 

#### 4.1普通混合

**会将混合内容输出到css文件中**

```less
.font_hn{
  color: red;
  font-family: microsoft yahei, "黑体", Arial, Simsun, "Arial Unicode MS", Mingliu, Helvetica;
}
h1{
  font-size: 28px;
  .font_hn;
}
```

#### 4.2不带输出的混合

 **加()后就不会在css中输出混合内容了** ，即输出的css文件中无font_hn

```less
.font_hn(){
  color: red;
  font-family: microsoft yahei, "黑体", Arial, Simsun, "Arial Unicode MS", Mingliu, Helvetica;
}
h1{
  font-size: 28px;
  .font_hn;
}
```

#### 4.3带参数的混合

类似函数的调用方式，参数名开头为@ 

```less
.border(@color){
  border: 1px solid @color;
}
h1{
  &:hover{
    .border(green);
  }
}
h2{
  &:hover{
    .border(#000);
  }
}

其css文件为：
h1:hover {
  border: 1px solid #008000;
  border: 21px #008000 #ff0000;
}
h2:hover {
  border: 1px solid #000000;
  border: 21px #000000 #ff0000;
}
```

#### 4.4带参数且有默认值的混合

使用 : 为参数添加默认值， 有了默认值，我们可以不用设置属性值也能被调用 

```less
.border_you(@color:red){
  border: 1px solid @color; 
}
h1{
  &:hover{
    .border_you();
  }
}
h2{
  &:hover{
    .border_you(yellow);
  }
}

其编译后的css文件如下：

/*带参数并且有默认值的混合*/
h1:hover {
  border: 1px solid #ff0000;
}
h2:hover {
  border: 1px solid #ffff00;
}
```

####  4.5命名参数（传实参时可以指定值给的是哪个参数） 

 引用mixin时可以通过参数名称而不是参数的位置来为mixin提供参数值。任何参数都以用过它的名称来使用，这样就不必按照任意特定的顺序来使用参数 

```less
.mixin(@color: black; @margin: 10px; @padding: 20px) {
  color: @color;
  margin: @margin;
  padding: @padding;
}

.class1 {
  .mixin(@margin: 20px; @color: #33acfe);
}
.class2 {
  .mixin(#efca44; @padding: 40px);
}

其编译后的css文件如下:
.class1 {
  color: #33acfe;
  margin: 20px;
  padding: 20px;
}
.class2 {
  color: #efca44;
  margin: 10px;
  padding: 40px;
}
```

#### 4.6匹配模式

 根据传入的参数来改变混合的默认呈现

```less
.border(all,@w: 5px){
  border-radius: @w;
}
.border(t_l,@w:5px){
  border-top-left-radius: @w;
}
.border(t_r,@w:5px){
  border-top-right-radius: @w;
}
.border(b-l,@w:5px){
  border-bottom-left-radius: @w;
}
.border(b-r,@w:5px){
  border-bottom-right-radius: @w;
}

footer{
  .border(t_l,10px);
  .border(b-r,10px);
  background: #33acfe;
}

其编译后的css文件如下：
footer {
  border: 21px t_l 10px;
  border-top-left-radius: 10px;
  border: 21px b-r 10px;
  border-bottom-right-radius: 10px;
  background: #33acfe;
}
```

#### 4.7@arguments变量

 @arguments代表所有的可变参数 

**注意事项：**

1. @arguments代表所有可变参数，参数的先后顺序就是你的（）括号内的参数的先后顺序

2. 在使用的赋值，值的位置和个数也是一一对应的，只有一个值，把值赋值给第一个，两个值，赋值给第一个和第二个，三个值赋值给第三个……以此类推，但是需要注意的是假如我想给第一个和第三个赋值，你不能写（值1，，值3），必须把原来的默认值写上去！

```less
.border(@x:solid,@c:red){
  border: 21px @arguments;
}
.div1{
  .border(solid);
}

其编译后的css文件为：
.div1 {
    border: 21px solid #ff0000;
}
```

#### 4.8混合的返回值

```less
.average(@x, @y) {
  @average: ((@x + @y) / 2);
  @he: (@x + @y);
}

div {
  .average(16px, 50px);
  padding: @average;
  margin: @he;
}

其编译后的css文件如下：
div {
  padding: 33px;
  margin: 66px;
}
```



### 5.运算

#### 5.1数值运算

 只需要在其中的一个数值上加上单位，其他单位由less自动加上 

```less 
.wp{
  width: 450px + 450;
  height: 400 + 400px;
}
```

#### 5.2颜色运算

 Less在运算时，先将颜色值转换为rgb模式，然后再转换为16进制的颜色值并且返回 

```less
.content{   Background:#000000 + 21;  }

Css文件编译结果
.content{  background:#212121;  }
```

### 6.函数

#### 6.1RGB函数

```less
.bgcolor{ background:rgb(0,133,0); }

Css结果：
.bgcolor{ background:#008500; }
```

#### 6.2Convert函数

```less
body{ width:convert(20cm,px); }  

Css文件如下:
body{ width:755.90551181px; }
```

### 7.命名空间

多人协作时，避免选择器重名问题，引入命名空间的概念，以#开头代表命名空间，#namespace>.selector即可调用

```less
#mynamespace() {
  background: #ffffff;
  .a{
    &:hover{
      color: #ff6600;
    }
    .b{
      background: #ff0000;
     }
   }
}

.bgcolor{
    #mynamespace>.a;
}
.bgcolor2{
	#mynamespace>.a>.b;
}

编译后css文件
.bgcolor {
  color: #888888;
}
.bgcolor:hover {
  color: #ff6600;
}
.bgcolor .b {
  background: #ff0000;
}
.bgcolor2 {
  background: #ff0000;
}
```

### 8.避免编译

将要避免编译的值用 "" 包裹起来，并在前面加~

```less
.class {
    filter: ~"ms:alwaysHasItsOwnSyntax.For.Stuff()";
}
Css文件如下:
.class {
    filter: ms:alwaysHasItsOwnSyntax.For.Stuff();
}
```

### 9.继承(extent)

性能比混合高
继承不支持带参数，灵活度比混合低 

他将**所放置它的选择器**与**匹配引用的选择器**进行合并。 

```less
a { // a 所放置它的选择器
    background-color: #fff;
    &:extend(.b); // .b匹配引用的选择器
    border-bottom: 2px;
}
.b {
    font-weight: 700;
    color: yellow;
}

编译后css文件
a {
  background-color: #fff;
  border-bottom: 2px;
}
.b, a {
  font-weight: 700;
  color: yellow;
}
```

还可以在选择器后加 :extend(.selector, .selector 可选值all)表示继承，在**括号内用逗号分隔多个继承元素**，若有all可选值，hover等伪类也会继承

### 10.条件表达式

```less
当我们想根据表达式进行匹配，而非根据值和参数匹配时，导引就显得非常有用。如果你对函数式编程非常熟悉，那么你很可能已经使用过导引。

为了尽可能地保留CSS的可声明性，LESS通过导引混合而非if/else语句来实现条件判断，因为前者已在@media query特性中被定义。

以此例做为开始：

.mixin (@a) when (lightness(@a) >= 50%) {
  background-color: black;
}
.mixin (@a) when (lightness(@a) < 50%) {
  background-color: white;
}
.mixin (@a) {
  color: @a;
}
when关键字用以定义一个导引序列(此例只有一个导引)。接下来我们运行下列代码：

.class1 { .mixin(#ddd) }
.class2 { .mixin(#555) }
就会得到：

.class1 {
  background-color: black;
  color: #ddd;
}
.class2 {
  background-color: white;
  color: #555;
}
导引中可用的全部比较运算有： > >= = =< <。此外，关键字true只表示布尔真值，下面两个混合是相同的：

.truth (@a) when (@a) { ... }
.truth (@a) when (@a = true) { ... }
除去关键字true以外的值都被视示布尔假：

.class {
  .truth(40); // Will not match any of the above definitions.
}
导引序列使用逗号‘,’—分割，当且仅当所有条件都符合时，才会被视为匹配成功。

.mixin (@a) when (@a > 10), (@a < -10) { ... }
导引可以无参数，也可以对参数进行比较运算：

@media: mobile;

.mixin (@a) when (@media = mobile) { ... }
.mixin (@a) when (@media = desktop) { ... }

.max (@a, @b) when (@a > @b) { width: @a }
.max (@a, @b) when (@a < @b) { width: @b }
最后，如果想基于值的类型进行匹配，我们就可以使用is*函式：

.mixin (@a, @b: 0) when (isnumber(@b)) { ... }
.mixin (@a, @b: black) when (iscolor(@b)) { ... }
下面就是常见的检测函式：
    iscolor
    isnumber
    isstring
    iskeyword
    isurl
如果你想判断一个值是纯数字，还是某个单位量，可以使用下列函式：
    ispixel
    ispercentage
    isem
最后再补充一点，在导引序列中可以使用and关键字实现与条件：
.mixin (@a) when (isnumber(@a)) and (@a > 0) { ... }

使用not关键字实现或条件
.mixin (@b) when not (@b > 0) { ... }
```

