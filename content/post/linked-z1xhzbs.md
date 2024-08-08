---
title: 链表
slug: linked-z1xhzbs
url: /post/linked-z1xhzbs.html
date: '2024-07-18 16:17:21+08:00'
lastmod: '2024-08-08 12:54:47+08:00'
toc: true
tags:
  - 链表
categories:
  - 数据结构
keywords: 链表
isCJKLanguage: true
---

# 链表

# 单向链表

```C++
#include <forward_list>
push_front() // 无push_back()
emplace_front()
front() // 无back()

int size = std::distance(std::begin(v), std::end(v));
```

## 成员函数

不支持size()

|函数|功能|
| --------------------| ------------------------------------|
|before\_begin()|返回指向第一个元素之前位置的迭代器|
|push\_front()|头部插入|
|pop\_front()|头部删除|
|insert\_after()|插入元素|
|erase\_after()|删除指定元素|

# 双向链表

```C++
#include <list>

# 创建一个有十个元素的list对象，每个元素都为4
list<int> l(10,4);
```

## 成员函数

|函数|功能||
| ------------| ----------| --|
|push_back|尾部插入||
|push_front|头部插入||
|pop_back|尾部删除||
|pop_front|头部删除||

## 剪切splice

```C++
# 将it指向的元素剪切到x的position上去，不会导致迭代器it失效
void splice ( iterator position, list<T,Allocator>& x, iterator it );
```
