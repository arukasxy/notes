---
title: stack栈
slug: stack-stack-zfgeku
url: /post/stack-stack-zfgeku.html
date: '2024-07-18 16:23:17+08:00'
lastmod: '2024-08-08 12:52:58+08:00'
toc: true
tags:
  - 栈
categories:
  - 数据结构
keywords: 栈
isCJKLanguage: true
---

# stack栈

# 栈

特点：

* 先入后出

```c++
#include <stack>
stack<int> st;
```

## 底层实现

支持empty()、back()、push_back()、pop_back()的容器

底层容器：vector、deque（默认）、list

## 成员函数

|函数|功能||
| -----------------------| ------| --|
|push()|压栈||
|pop()|弹栈||
|top()|栈顶||
|stack<int>().swap(st)|清空||

## 出栈序列

* n个不同元素进栈，出栈序列的个数为$1/(n+1)*\mathrm{C}_{2n}^{n}$

# 双栈

​![image](https://raw.githubusercontent.com/arukasxy/notes/main/content/post/image-viewer/image-20240718202612-qi5ago7.png)​

特点：

* 用一个数组存储两个栈
* 场景：  
  当两个栈的空间需求有相反的关系时；  
  最好是一个栈增长时，另一个栈缩短；  
  需要从一个栈中取出元素放入另一个栈
