---
title: Java的字符串操作
date: 2020-07-31 17:04:16
tags:
  - Java
categories:
  - 大学课程
---

## 一、比较

| 语言  | 操作          | 形式                                                         | 机制                                                         | 线程安全性               |
| ----- | ------------- | ------------------------------------------------------------ | ------------------------------------------------------------ | ------------------------ |
| c/c++ | char*         | 字符指针                                                     | 通过手动修改指针指向的内存空间修改字符串                     | 未知                     |
| c/c++ | String        | 容器类                                                       | 内部使用char数组存储字符，但内存管理，分配和null终止都由字符串类本身来处理 | 并发的读操作是线程安全的 |
| Java  | String        | 1. String a = "a"，是以字面常量的形式储存在常量池中<br />2. new String(“a”) 创建的以对象的形式存放在堆中 | **对String对象的任何改变都不影响到原对象，相关的任何change操作都会生成新的对象** | 安全                     |
| Java  | StringBuilder | 对象                                                         | **所有操作都在原有的对象上进行**                             | 不安全                   |
| Java  | StringBuffer  | 对象                                                         | **所有操作都在原有的对象上进行**                             | 安全                     |

<!--more-->

## 二、性能测试

### 测试代码

```java
public class Main {

    private static int NUM = 100000;
    
    public static void main(String[] args) {
        System.out.println(NUM + ":");
        testString();
        testStringBuffer();
        testStringBuilder();
    }

    private static void testString() {
        long start = System.currentTimeMillis();
        String str = "";
        for (int i = 0; i < NUM; i++) {
            str += i;
        }
        long end = System.currentTimeMillis();
        System.out.println("String:" + (end - start) + "ms");
    }

    private static void testStringBuffer() {
        long start = System.currentTimeMillis();
        StringBuffer str = new StringBuffer();
        for (int i = 0; i < NUM; i++) {
            str.append(i);
        }
        long end = System.currentTimeMillis();
        System.out.println("StringBuffer:" + (end - start) + "ms");
    }

    private static void testStringBuilder() {
        long start = System.currentTimeMillis();
        StringBuilder str = new StringBuilder();
        for (int i = 0; i < NUM; i++) {
            str.append(i);
        }
        long end = System.currentTimeMillis();
        System.out.println("StringBuilder:" + (end - start) + "ms");
    }
}
```

### 测试结果

```reStructuredText
100000:
String: 2907ms
StringBuffer: 9ms
StringBuilder: 5ms
```

在该测试中String在字符串拼接上明显低于其他两者，也说明了String每次拼接字符串都要新建对象的时间消耗很大，而StringBuffer加了synchronized是线程安全的，在效率上不如StringBuilder，符合了预期。

## 三、java正则表达式

### 代码

```java
public static void main(String[] args) {
    // 邮政编码
    String postal_code = "[1-9]\\d{5}";
    // 区号-座机号码
    String landline = "\\d{3}-\\d{8}|\\d{4}-\\d{7}";
    // 手机号码
    String phone = "1[345678]\\d{9}";

    // 测试字符串
    String text = "513215 13411156663 010-88888888";

    Pattern r = Pattern.compile(postal_code);
    Matcher m = r.matcher(text);
    System.out.println("邮政编码：");
    if (m.find()) {
        System.out.println(m.group());
    }

    r = Pattern.compile(landline);
    m = r.matcher(text);
    System.out.println("区号-座机号码：");
    if (m.find()) {
        System.out.println(m.group());
    }

    r = Pattern.compile(phone);
    m = r.matcher(text);
    System.out.println("手机号码：");
    if (m.find()) {
        System.out.println(m.group());
    }
}
```

### 结果

```reStructuredText
邮政编码：
513215
区号-座机号码：
010-88888888
手机号码：
13411156663
```

## 参考资料

[探秘Java中的String、StringBuilder以及StringBuffer](https://www.cnblogs.com/dolphin0520/p/3778589.html)
[C++ string class and its applications](https://www.geeksforgeeks.org/c-string-class-and-its-applications/)
[https://www.runoob.com/java/java-regular-expressions.html](https://www.runoob.com/java/java-regular-expressions.html)
