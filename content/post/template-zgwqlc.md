---
title: 模板
slug: template-zgwqlc
url: /post/template-zgwqlc.html
date: '2024-07-12 16:41:44+08:00'
lastmod: '2024-08-08 13:03:52+08:00'
toc: true
tags:
  - 模板
categories:
  - 语法
keywords: 模板
isCJKLanguage: true
---

# 模板

特点：

* 作用：**减少重复代码的编写**

# 非类型参数模板

特点：

1. 仅能使用int、char、short等整数，不能使用浮点数或字符串
2. 作用：将**常量值**作为模板参数
3. 常用于数据空间动态定义等场景

```C++
template<int radix>
bool convert(char* buf, size_t bufsize, int value);
```

# 函数模板

特点：

1. 本质为函数重载
2. 编译时多态，无运行时开销
3. 常用于处理**不同数据类型**的场景
4. 可以自动类型推导

可变参数模板参考可变参数

```C++
template<class T=int, typename Alloc>
class Swa(T a,T b)
{
    T temp;
    temp = a;
    a = b;
    b = temp;
};
```

# 类模板

* 无自动类型推导，只能显式指定类型

```C++
template <class T=int, typename Alloc>
class complex
{
public:
	complex(T r = 0, T i=0):re(r), im(i){}
}
```

# 模板特化

```C++
# 完全特化
template<>
class模板名<int>{
	...
};

# 偏特化
template<class Alloc>
class vector<bool,Alloc>{
	...
};
```

# 完美转发（+引用折叠）

作用：函数既可以传递左值，也可以传递右值

原理：引用折叠

* 当实参为左值或者左值引用（A&）时，函数模板中 T&& 将转变为 A&（A& && = A&）
* 当实参为右值或者右值引用（A&&）时，函数模板中 T&& 将转变为 A&&（A&& && = A&&）

```C++
template <typename T>
void function(T&& t) {
    otherdef(forward<T>(t));
}
```

‍
