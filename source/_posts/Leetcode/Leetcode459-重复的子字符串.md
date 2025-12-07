---
title: Leetcode459. 重复的子字符串
date: 2021-01-23 22:35:12
tags:
  - leetcode
categories:
  - 算法
---
> 题目链接: https://leetcode-cn.com/problems/repeated-substring-pattern/

给定一个非空的字符串，判断它是否可以由它的一个子串重复多次构成。给定的字符串只含有小写英文字母，并且长度不超过10000。
<!--more-->
## 俺愚蠢的暴力解法

```js
var repeatedSubstringPattern = function(s) {
    for (let i = 1; i <= s.length / 2 && i < 10000; i++) {
        let str = s.slice(0, i);
        let repeatNum = s.length / str.length;
        if (Number.isInteger(repeatNum) && str.repeat(repeatNum) === s) {
            return true;
        }
    }
    return false
};
```

![image-20210123215828920](image-20210123215828920.png)

## 方法二：字符串匹配

> 牛啊，官方题解的证明方式没看懂，之后有时间回来看看

### 算法

1. 用两个`s`首尾相连得到一个新的字符串`ss`;
2. 去掉`ss`的首尾两个字符；
3. 如果在剩下来的字符串中能找到`s`那么返回True，否则False

```js
var repeatedSubstringPattern = function(s) {
    return (s+s).slice(1, -1).indexOf(s) !== -1;
};
```

![image-20210123223744746](image-20210123223744746.png)

### 证明方式

https://blog.csdn.net/qq_23997101/article/details/78804826

## 方法三：正则表达式

```js
var repeatedSubstringPattern = function(s) {
    return /^([a-z]+)\1+$/.test(s)
};
```

![image-20210123222502223](image-20210123222502223.png)

不解释，草。
