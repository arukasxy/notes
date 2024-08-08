---
title: 元组
slug: turtle-1geggg
url: /post/turtle-1geggg.html
date: '2024-07-18 17:12:05+08:00'
lastmod: '2024-08-08 10:30:19+08:00'
toc: true
isCJKLanguage: true
tags:
  - tuple
  - pair
keywords: tuple,pair
description: 存储多个变量
---

# 元组

* STL通常没有实现对应的哈希函数，因此在使用unordered_map/unordered_set时不能直接使用

# pair对值

```C++
#include <utility>
std::pair<int, char> mypair(std::make_pair(20, 'b'));

mypair.first;
mypair.second;

# 接收pair对象
std::tie(name, ages) = func();
auto [a,b] = make_pair(2, 3);
```

# tuple元组

```C++
#include <tuple>
std::tuple<int, char> mytuple(std::make_tuple(20, 'b'))

# 接收tuple对象
std::get<0>(mytuple) = 100;
auto [a, b, c] = mytuple;
std::tie(a, b, c) = std::make_tuple("Tom", 20, 1.75);
```

# 展开为函数参数

可变参数模板参考可变参数，需C++17支持

```C++
# 配合函数使用
int sum(int a, int b, int c) {
    return a + b + c;
}
std::tuple<int,int,int> t(1,2,3);
# sum为可调用对象
int res = std::apply(sum, std::move(t));
```

```C++
# 配合可变参数模板使用，注意使用lambda封装，使用auto自动类型推导
#include <iostream>
#include <tuple>
#include <utility>
using namespace std;

template<typename T>
T sum(T t) {
    return t;
}

template<typename T, typename... Types>
T sum(T first, Types... rest) {
    return first + sum<T>(rest...);
}

int main(){
  std::tuple<int,int,int> t(1, 2, 3);
  cout<< std::apply([](auto &&... args) {return sum(args...);},std::move(t))<<endl;
  return 0;
}
```

```C++
# 配合仿函数使用
struct TT {
    int sum(int a, int b, int c) {
        return a + b + c;
    }
};

int main(int argc, const char *argv[]) {
    std::tuple tu(TT(), 1, 2, 3); // 第一个成员就是调用成员
    int res = std::apply(&TT::sum, std::move(tu)); // 这里传成员函数指针
    // 等同于std::get<0>(tu).sum(1, 2, 3)
    std::cout << res << std::endl;
    return 0;
}
```

```C++
# 配合匿名函数使用
std::tuple<int, std::string, float> t1(10, "Test", 3.14);
std::apply([](auto&&... args) {
    ((std::cout << args << '\n'), ...);
}, t1);
```

## 展开为构造函数参数

```C++
class Test {
public:
    Test(int a, double b, const std::string &c): a_(a), b_(b), c_(c) {}
    void show() const {std::cout << a_ << " " << b_ << " " << c_ << std::endl;}
private:
    int a_;
    double b_;
    std::string c_;
};

// 自行封装构造过程
template <typename T, typename... Args>
T Create(Args &&...args) {
    return T(args...);
}

int main(int argc, const char *argv[]) {
    std::tuple tu(1, 2.5, "abc");

# 1. 使用make_form_tuple
	Test &&t = std::make_from_tuple<Test>(std::move(tu));

# 2. 使用自己的封装结构 Create
    Test &&t = std::apply([](auto &&...args)->Test {return Create<Test>(args...);}, std::move(tu));

    t.show(); // 打印：1 2.5 abc
    return 0;
}

```
