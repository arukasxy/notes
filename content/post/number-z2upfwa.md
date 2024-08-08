---
title: 数字
slug: number-z2upfwa
url: /post/number-z2upfwa.html
date: '2024-07-14 12:05:30+08:00'
lastmod: '2024-08-08 12:53:27+08:00'
toc: true
tags:
  - 数字
categories:
  - 数据结构
keywords: 数字
isCJKLanguage: true
---

# 数字

# 大小范围（数值极限）

|数据类型|大小|头文件|
| ------------------------| -----------------| ----------|
|INT_MAX<br />INT_MIN|2147483647<sup>(10位)</sup><br />-2147483648<br />|limits.h|
|FLT_MAX<br />FLT_MIN||limits.h|
|DBL_MAX<br />DBL_MIN|<br />|float.h|
|LONG_MAX<br />LONG_MIN<br />|||
|LLONG_MAX<br />LLONG_MIN<br />|||

float能表示的最大正整数为2^32

double能表示的最大正整数为2^64

```c++
#include <limits>
using PageNum = int32_t;
std::numeric_limits<PageNum>::max()
std::numeric_limits<short>::max()
numeric_limits<char>::is_signed  // 是否带符号
numeric_limits<string>::is_specialized  // 是否存在limit
```

# 空间大小(+数字类型)

|数字类型|32位|64位|
| --------------------| -------| ------|
|char|1字节||
|short|2||
|int|4||
|float|4||
|double|8||
|long|4|8|
|long long|8||
|unsigned long long|8||

# 进制转换

也可以使用stringstream实现

```C++
# 任意进制转十进制
int a = stoi(str, 0, 2);
int stoi(int m, string s){ 
    int ans=0;
    for(int i=0;i<s.size();i++){
        char t=s[i];
        if(t>='0'&&t<='9') 
			ans=ans*m+t-'0';
        else 
			ans=ans*m+t-'a'+10;
    }
    return ans;
}

# 十进制数n转任意m进制
#include<cstdlib>
char *itoa(int value, char *string, int radix);
itoa(num, str, 2);
string itoa(int n,int m){
    string ans="";
    do{
        int t=n%m;
        if(t>=0&&t<=9)  
			ans+=(t+'0');
        else 
			ans+=(t+'a'-10);
        n/=m;
    }while(n);   
    reverse(ans.begin(),ans.end());
    return ans;  
}
```

# 随机数生成rand

要取得[a,b)的随机整数，使用(rand() % (b-a))+ a （结果值含a不含b）

要取得[a,b]的随机整数，使用(rand() % (b-a+1))+ a （结果值含a和b）

要取得(a,b]的随机整数，使用(rand() % (b-a))+ a + 1 （结果值不含a含b）

```C++
#include<time.h>
#include<cstdlib>

# 设置随机种子，time(nullptr) 一秒钟才变化一次，作为随机种子变化频率太低，容易被预测
srand(time(NULL));

# 生成[0,i)之间的随机数
#define random(x) (rand()%x)

# 使用静态变量保存状态，调用时加锁，不可重入
int rand(void);
# rand的可重入版本
int rand_r(unsigned int *seedp);
```

# 判断两个数是否异号

```C++
int x = -1, y = 2;
bool f = ((x ^ y) < 0); // true
```

# 判断是否是2的指数

原理：一个数如果是 2 的指数，那么它的二进制表示一定只含有一个 1

```C++
bool isPowerOfTwo(int n) {
    if (n <= 0) return false;
    return (n & (n - 1)) == 0;
}
```

# 数字类型大小size_t

```C++
#include <cstddef>
```

‍
