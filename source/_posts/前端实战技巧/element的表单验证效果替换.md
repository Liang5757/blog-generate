---
title: element的表单验证效果替换
date: 2020-09-15 22:17:25
tags:
  - CSS
  - 技巧
categories:
  - 前端
---

在做项目的时候看到了这个需求，需要用图片替换默认的文字显示

![image-20200915220854381](image-20200915220854381.png)
<!--more-->
f12可以看到出错了的表单项会在`div.el-form-item__content`下有个`div.el-form-item__error`

![image-20200915221210128](image-20200915221210128.png)

于是就对`div.el-form-item__error`进行操作

```scss
/deep/ .el-form-item__content {
    display: flex;	// 使错误信息显示在同一行
    margin-right: 20px;

    // 修改错误显示
    /deep/ .el-form-item__error {
        position: relative;
        top: 0;
        right: 0;
        font-size: 0;	// 把默认文字取消

        // 用伪元素插入背景图片
        &:after {
            content: "";
            display: block;
            width: 20px;
            height: 20px;
            position: absolute;
            right: -10px;
            top: 10px;
            background: url("~@/assets/img/error.png") 0 0 no-repeat;
            background-size: 20px 20px;
        }
    }
}
```


