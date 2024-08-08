---
title: map字典
slug: map-dictionary-z1xnnbh
url: /post/map-dictionary-z1xnnbh.html
date: '2024-07-18 16:23:58+08:00'
lastmod: '2024-08-08 12:32:33+08:00'
toc: true
tags:
  - 字典
categories:
  - 数据结构
keywords: 字典
isCJKLanguage: true
---

# map字典

# 成员函数

|函数|功能||
| ---------| ------| --|
|empty()|判空||
||||

# map

* 底层实现：红黑树，查找时间复杂度为$O(logn)$
* 按key进行排序，默认std::less从小到大
* key可以使用pair

```c++
#include <map>

# 从大到小排序
map<string, int, std::greater<string>> mapWord;

# 删除数据
mp.erase(key)
if(iter != mp.end())
    iter = mp.erase (iter) ;
```

# unordered_map

* 底层实现：哈希表
* 查询复杂度为O(1)，适用于查找频率高
* key不能使用pair，需自定义哈希函数
* 不能删除指定元素

```c++
#include <unordered_map>

unordered_map<int, string> mp = {{1, "apple"}, {2,"watermelon"}};

# 添加数据
mp.insert(unordered_map<int,string>::value_type(1,”sakura”))
mp.insert(std::pair<int,string>(1,”sakura”))
mp.insert(std::make_pair(1,”sakura”))
people["Jim"] = 22;

# 删除元素
mp.erase(key) //返回删除元素的个数
if(iter != mp.end())
    iter = mp.erase (iter) ;	//返回被移除元素后的元素的迭代器
```

# multimap

* 底层实现：红黑树，查找时间复杂度为$O(logn)$

* 允许key重复

```c++
#include <multimap>
```

‍
