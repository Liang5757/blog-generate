---
title: WordCount (Python)
date: 2020-07-28 21:44:44
tags:
  - 软件工程
categories:
  - 大学课程
---
# WordCount (Python)

### Github项目地址：https://github.com/w1036933220/WordCount
<!--more-->
## 一、解题思路

1. 把项目需求理清楚，画一个思维导图

![image-20200318234358738](image-20200318234358738.png)

2. 考虑各部分功能所需要的大概实现思路

![image-20200318234452134](image-20200318234452134.png)

3. 然后完成了计算文件属性的算法部分
4. 再回头想对指令的解析问题，顺带添加了递归处理多个文件的功能
5. 查python的os库文档，最后决定用os.walk读取当前文件夹内的所有文件夹和文件，替换掉输入的*和?通配符，再进行匹配

## 三、设计实现过程及代码说明

1. main.py（入口文件）

```python
from utils.utils import *

if __name__ == '__main__':
    command = input("请输入命令(wc.exe [parameter] {file_name}):\n")

parse_command(command)
```

2. orders.py
	存放指令集，和输出各类数据的函数

```python
from utils.count import *
from views.main_view import *
import tkinter as tk


# 输出字符数
def print_char_num(text):
	print("字符数：" + str(FileProperties(text).count_char_num()))


# 输出单词数
def print_word_num(text):
	print("词数：" + str(FileProperties(text).count_word_num()))


# 输出行数
def print_line_num(text):
	print("行数：" + str(FileProperties(text).count_line_num()))


# 输出代码行/空行/注释行
def print_code_property(text):
    file_properties = FileProperties(text)

    print("空行：" + str(file_properties.count_null_line_num()))
    print("注释行：" + str(file_properties.count_annotation_line_num()))
    print("代码行：" + str(file_properties.count_code_line_num()))


# 调出图形界面
def draw_view():
	root = MainView(tk.Tk())



def print_error():
	print("指令输入错误")


# 单指令命令集
orders = {
    '-c': print_char_num,

    '-w': print_word_num,

    "-l": print_line_num,

    "-a": print_code_property,

    "-s": print_error,

    "-x": draw_view
}
```

3. utils.py
	放置解析指令、读取文件、模糊搜素的函数

```python
import os
from utils.orders import *


# description：parse command
# param：order input
# return：[order, [file_list]] / FALSE
def parse_command(command):
    command = command.strip().split(" ")

    # 指令若为空或者起始不为wc.exe则报错
    if command == [] or command[0] != "wc.exe":
        print("指令输入错误")

    # 打开图形界面的指令(一级指令)
    if "-x" in command:
        orders.get("-x")
    elif len(command) > 2:
        order = command[-2]
        file_name = command[-1]
        file_list = get_file_list(file_name)

        # 递归调用的指令
        if "-s" in command:
            if file_list:
                for file in file_list:
                    print(file + ":")
                    text = read_file(file)
                    orders.get(order)(text)
        else:
            print(file_list[0] + ":")
            text = read_file(file_name)
            orders.get(order)(text)
    else:
        print("指令输入错误")


# 读取目录下符合条件的文件名
def get_file_list(file_name):
    # 最终构建的文件列表
    file_list = []
    # 匹配到的文件夹列表、需二次处理
    dir_list = []

    file_name = file_name.replace("?", "\\S").replace("*", "\\S+")
    file_name += "$"

    for root, dirs, files in os.walk(".", topdown=False):
        for name in files:
            if re.match(file_name, name):
                file_list.append(os.path.join(root, name))
        for name in dirs:
            if re.match(file_name, name):
                dir_list.append(os.path.join(os.getcwd() + os.sep, name))

    # 如果文件夹非空，则继续收集
    if dir_list:
        for item in dir_list:
            all_file = os.listdir(item)
            for file in all_file:
                # 文件的完整路径
                file_path = item + os.sep + file
                if os.path.isfile(file_path):
                    file_list.append(file_path)

    print(file_list)

    return file_list


# description：read files
# param：file_list
# return：file content
def read_file(file):
    with open(file, 'r') as f:
        return f.readlines()
```

4. count.py
	存放计算文件属性的类

```python
import re

class FileProperties(object):

    def __init__(self, file_text):
        self.file_text = file_text
        # 字符数
        self.char_num = 0
        # 单词数
        self.word_num = 0
        # 行数
        self.line_num = len(file_text)
        # 空行
        self.null_line_num = 0
        # 代码行
        self.code_line_num = 0
        # 注释行数
        self.annotation_line_num = 0

    # 计算字符数
    def count_char_num(self):
        for line in self.file_text:
            self.char_num += len(line.strip())

        return self.char_num

    # 计算单词数
    def count_word_num(self):
        for line in self.file_text:
            # 正则匹配一行中的所有单词，并计算单词数
            self.word_num += len(re.findall(r'[a-zA-Z0-9]+[\-\']?[a-zA-Z]*', line))

        return self.word_num

    # 计算行数
    def count_line_num(self):

        return self.line_num

    # 计算空行数
    def count_null_line_num(self):
        for line in self.file_text:
            # 只有不超过一个可显示的字符
            if len(re.findall(r'\S', line)) <= 1:
                self.null_line_num += 1

        return self.null_line_num

    # 计算代码行
    def count_code_line_num(self):
        return self.line_num - self.null_line_num - self.annotation_line_num

    # 计算注释行
    def count_annotation_line_num(self):
        flag = 0
        for line in self.file_text:
            line = line.strip()
            # 匹配不是代码行且有//
            if re.match(r'^\S?\s*?\/\/', line):
                self.annotation_line_num += 1
            # 匹配不是代码行且有/*
            elif re.match(r'^\S?\s*?\/\*', line):
                flag = 1
                self.annotation_line_num += 1
                if line.endswith('*/'):
                    flag = 0
            elif flag == 1:
                self.annotation_line_num += 1
            elif "*/" in line:
                self.annotation_line_num += 1
                flag = 0

        return self.annotation_line_num
```

5.main_view.py(新增)

```python
import tkinter as tk
from tkinter import ttk
import tkinter.filedialog
from utils.utils import *
from utils.count import *


class MainView(object):

    def __init__(self, window):
        self.window = window
        self.window.title("这是船新的版本！")
        self.window.geometry("540x290")
        self.data_tree = ttk.Treeview(self.window, show="headings")
        self.creat_view()

    def creat_view(self):
        # 选择文件按钮
        btn = tk.Button(self.window, text="选择文件", command=self.file_choose).place(x=240, y=247)

        # 文件数据显示表格
        self.data_tree.place(x=8, y=8)
        # 定义列
        self.data_tree["columns"] = ("文件名", "字符数", "单词数", "行数", "空行数", "代码行数", "注释行数")
        # 设置列属性，列不显示
        self.data_tree.column("文件名", width=100)
        self.data_tree.column("字符数", width=70)
        self.data_tree.column("单词数", width=70)
        self.data_tree.column("行数", width=70)
        self.data_tree.column("空行数", width=70)
        self.data_tree.column("代码行数", width=70)
        self.data_tree.column("注释行数", width=70)
        # 设置表头
        self.data_tree.heading("文件名", text="文件名")
        self.data_tree.heading("字符数", text="字符数")
        self.data_tree.heading("单词数", text="单词数")
        self.data_tree.heading("行数", text="行数")
        self.data_tree.heading("空行数", text="空行数")
        self.data_tree.heading("代码行数", text="代码行数")
        self.data_tree.heading("注释行数", text="注释行数")

        self.window.mainloop()

    def file_choose(self):
        file_list = tk.filedialog.askopenfilenames()
        for index, file in enumerate(file_list):
            text = read_file(file)
            [char_num, word_num, line_num, null_line_num, code_line_num,
             annotation_line_num] = FileProperties(text).all_count()
            file = file.split("/")[-1]
            self.data_tree.insert('', index, values=(file, char_num, word_num, line_num,
                                                 null_line_num, code_line_num, annotation_line_num))
```

## 五、PSP表格

| PSP2.1                                  | Personal Software Process Stages        | 预估耗时（分钟） | 实际耗时（分钟） |
| --------------------------------------- | --------------------------------------- | ---------------- | ---------------- |
| Planning                                | 计划                                    | 10               | 8                |
| · Estimate                              | · 估计这个任务需要多少时间              | 10               | 8                |
| Development                             | 开发                                    | 460              | 610              |
| · Analysis                              | · 需求分析 (包括学习新技术)             | 120              | 200              |
| · Design Spec                           | · 生成设计文档                          | 90               | 60               |
| · Design Review                         | · 设计复审 (和同事审核设计文档)         | 5                | 5                |
| · Coding Standard                       | · 代码规范 (为目前的开发制定合适的规范) | 5                | 0                |
| · Design                                | · 具体设计                              | 120              | 100              |
| · Coding                                | · 具体编码                              | 90               | 200              |
| · Code Review                           | · 代码复审                              | 10               | 30               |
| · Test                                  | · 测试（自我测试，修改代码，提交修改）  | 20               | 15               |
| Reporting                               | 报告                                    | 30               | 32               |
| · Test Report                           | · 测试报告                              | 10               | 12               |
| · Size Measurement                      | · 计算工作量                            | 10               | 5                |
| · Postmortem & Process Improvement Plan | · 事后总结, 并提出过程改进计划          | 10               | 8                |
| 合计                                    |                                         | 355              | 630              |

## 六、测试运行

![image-20200324233229734](image-20200324233229734.png)

![image-20200324233138984](image-20200324233138984.png)

图形界面测试（新增）

![image-20200325013347708](image-20200325013347708.png)

![image-20200325020159689](image-20200325020159689.png)

## 七、总结

1. 太久没写python了，发现居然没有switch这个语句，百度查到了表驱动这个东西
2. 用时与预计的出入有点大
3. 输入验证处理没有做完全，图形界面没时间做了（过了大半年遗忘率确实高）
4. 写的时候发现python有些问题不知道是bug还是什么

```python
orders = {
    '-c': print_char_num,
    '-w': print_word_num,
    "-l": print_line_num,
    "-a": print_code_property,
    "-s": print_error,
}
```

这是我的指令集，如果把他写成像js

```javascript
orders = {
    '-c': function () {
    	// 某些操作
    },
}
```

python不管会不会用到这个指令集，都会把字典中的值执行一遍，所以只能放函数名，js就不会