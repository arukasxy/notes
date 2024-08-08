---
title: string字符串
slug: string-string-1g9olm
url: /post/string-string-1g9olm.html
date: '2024-07-10 23:42:16+08:00'
lastmod: '2024-08-08 13:01:41+08:00'
toc: true
tags:
  - 字符串
categories:
  - 数据结构
keywords: 字符串
isCJKLanguage: true
---

# string字符串

# 翻转字符串

```C++
reverse(s.begin(), s.end());
reverse(s.begin()+i , s.begin()+i+k); // 翻转以i为起始的k个字符串

void reverse(char *str, int n){
	std::reverse(str, str+n);
}
```

# 分割split

```c++
void SplitString(const std::string& s, std::vector<std::string>& v, const std::string& c)
{
  std::string::size_type pos1, pos2;
  pos2 = s.find(c);
  pos1 = 0;
  while(std::string::npos != pos2)
  {
	// if(pos2 != pos1) 忽略空子串
    v.push_back(s.substr(pos1, pos2-pos1));
 
    pos1 = pos2 + c.size();
    pos2 = s.find(c, pos1);
  }
  if(pos1 != s.length())
    v.push_back(s.substr(pos1));
}
```

使用stringstream：会保存空字符串，支持连续分隔符（中间为空字符串），支持字符串分隔符

```c++
#include <sstream>
stringstream ss(line) 
或 ss<<line;
# n个字符串
for(int i=0;i<n;i++)
	getline(ss,v[i],',');
# 不确定字符串数目
while(getline(ss,str,','))
	vec.push_back(str);
```

分隔符可能连续：

```c++
void SplitString(const std::string& str, std::vector<std::string>& res, const char chrc)
{
	if ( str.empty( ) ) return ;		//判空
	std::string strs = str + chrc;  //末尾加上分隔符方便计算
	size_t size = strs.size();
	size_t pos{ 0 };				//分割位置			
	bool meet{ false };             //分隔符连续标志位
 
	for (size_t i = 0; i <= size; i++)
	{
		if (strs[i] != chrc && !meet) // 不是分割符，分隔符不连续
		{
			pos = i;
			meet = true;
		}
		if (strs[i] == chrc && meet) // 是分隔符，无非分隔符数据
		{
			res.push_back(strs.substr(pos, i - pos));
			pos = i;
			meet = false;
		}
	}
	return ;
}
```

将结果保存在set中，分隔符为delim中任意一个字符，结果不包括delim，可支持连续分隔符：

```c++
void split_string(const std::string &str, std::string delim, std::set<std::string> &results)
{
  int cut_at;
  std::string tmp_str(str);
  while ((cut_at = tmp_str.find_first_of(delim)) != (signed)tmp_str.npos) {
    if (cut_at > 0) {
      results.insert(tmp_str.substr(0, cut_at));
    }
    tmp_str = tmp_str.substr(cut_at + 1);
  }

  if (tmp_str.length() > 0) {
    results.insert(tmp_str);
  }
}
```

将结果保存在vector中，分隔符为delim中任意一个字符，如果超内存，则可以用第一版本：

```c++
void split_string(const std::string &str, std::string delim, std::vector<std::string> &results)
{
  int cut_at;
  std::string tmp_str(str);
  while ((cut_at = tmp_str.find_first_of(delim)) != (signed)tmp_str.npos) {
    if (cut_at > 0) {
      results.push_back(tmp_str.substr(0, cut_at));
    }
    tmp_str = tmp_str.substr(cut_at + 1);
  }

  if (tmp_str.length() > 0) {
    results.push_back(tmp_str);
  }
}
```

C风格字符串分割，keep_null 是否保存空子字符串：

```c++
void split_string(char *str, char dim, std::vector<char *> &results, bool keep_null)
{
  char *p = str;
  char *l = p;
  while (*p) {
    if (*p == dim) {
      *p++ = 0;
      if (p - l > 1 || keep_null)
        results.push_back(l);
      l = p;
    } else
      ++p;
  }
  if (p - l > 0 || keep_null)
    results.push_back(l);
  return;
}
```

# 字符类型判断

字符以ASCII码存储，如‘0’的ASCII值为48，空字符的ASCII值为0

​![image](https://raw.githubusercontent.com/arukasxy/notes/main/content/post/image-viewer/image-20240716221425-pp6ex6k.png)​

# 追加（拼接）

单个字符使用+=，其余情况用append

|<br />|append|+=|push_back|
| ----------------| -----------------------------------------------------| -----------------| ----------------------|
|string|str.append(str2);|str1 += str2;|√|
|substring|str1.append(str2, 0, 5);<sup>(追加str2从下标0开始的5个字符)</sup>|×|×|
|字符串常量|str1.append("Geeks");|str += "Geeks";|×|
|char ch[6]|str1.append(ch);|str += ch;|×|
|char|×|str += 'C';|str1.push_back('C');|
|iterator range|str1.append(str2.begin() + 5,<br />      str2.end());<br />|×|×|

```c++
# 把 src 所指向的字符串追加到 dest 所指向的字符串的结尾
char *strcat(char *dest, const char *src)

# 将src的前n个字符拼接到dest中(确保dest有足够的内存空间)
char *strncat(char *dest, const char *src,int n)

# 使用sprintf拼接
char *a = "aaaaa";
char *b = "bbbbb";
sprintf(s,"%s%s",a,b);

# 拼接vector<string>
reduce(v.begin(), v.end());
```

# 插入

```c++
string& insert (size_t pos, const string& str, size_t subpos, size_t sublen)

# 在index位置插入字符
string::iterator it = str1.insert(str1.begin(),'a');

# 在index位置插入2个字符
string sstr = str.insert(0,2,'a');

# 在index位置插入字符串
string sstr = str.insert(1,"hello~");
string sstr = str.insert(1,"hello~",3); //hel 前count个字符
string sstr = str2.insert(6,str1,3,string::npos);
// 在位置6插入str1字符串从3开始的npos个字符
// npos为size_t的最大值
```

# 排序

如果字符串长度相同，按字典序排序

如果字符串长度不同，字符串长的越大

```c++
sort(s.begin(),s.end());

# 字典序排序规则
bool cmp (string &a, string &b){
	if(a.size()==b.size()) return  a>b;
	return a.size() > b.size();
}
```

# 大小

注意数组char[]作为参数，会退化为指针

```c++
#include<cstring>
s.length();
s.size();
string::max_size(); // 字符串大小上限

# 返回指针所占字节大小(和计算机位数有关)
sizeof(char* s)
# 返回字符数组所占字节大小(包含\0)
sizeof(char[] s) 

# 返回字符个数(不包含\0)
strlen(char[] s)
strlen(char* s)

size_t strlen(const char *str) {
    size_t length = 0;
    while (*str++)
        ++length;
    return length;
}
```

||sizeof|strlen|
| ------| --------------| ----------------------|
|时间|编译时|运行时|
|空间|所占字节大小|字符串长度|
|大小|包含\0|不包含\0|
||运算符|库函数|
|参数|任意类型|字符型指针且以\0结尾|

# 类型转换

## 数字转string/char*

与进制转换相关参考进制转换

```c++
std::to_string(3.1415926);

# 支持int32_t
#include <sstream>
template<typename T> string toString(const T& t){
    ostringstream oss;  //创建一个格式化输出流
    oss<<t;             //把值传递如流中
    return oss.str();   
}

# int 转 char*
char *intStr = itoa(10);
char *itoa(int i,char *s,int radix); // radix指定进制
```

## C-string 转string

```c++
char ch[2] = {'a'+5, 0};
string s(ch); // 仅支持char*，不支持char

#include <sstream>
char key = 'a'+5;
stringstream ss;
ss<<key;
string s;
ss>>s;
```

## string 转C-string

```c++
const char *c_str(); // 自动添加终止符
char arr[str.length() + 1]; 
strcpy(arr, str.c_str()); 
或
char a[20];
std::strcpy(a, std::data(s));
```

## string/char*转数字

与进制转换相关参考进制转换

```c++
# 将2进制字符串从位置0开始转换为十进制
int a = stoi(str, 0, 2);

# char* 转 int(十进制)
#include <cstdlib>
int a = atoi(s.c_str())

# char* 转 long
long a = strol(s.c_str(),nullptr,10) // 10进制

# char* 转 long long
long long a = stoll(s.c_str())

# char* 转 float
float a = stof(s.c_str())

# char* 转 double
double a = stod(s.c_str())
```

## 大小写转换

```c++
#include <cctype>
transform(out.begin(), out.end(), out.begin(), ::toupper); // 大写
transform(out.begin(), out.end(), out.begin(), ::tolower); // 小写
或
#include <string>
transform(s.begin(),s.end(),s.begin(),(int (*)(int))toupper);
transform(s.begin(),s.end(),s.begin(),(int (*)(int))tolower);

# 转小写（位运算）
('a' | ' ') = 'a'
('A' | ' ') = 'a'

# 转大写（位运算）
('b' & '_') = 'B'
('B' & '_') = 'B'

# 大小写互换
('d' ^ ' ') = 'D'
('D' ^ ' ') = 'd'
```

## 字符流（转任意类型）

```C++
#include <sstream>
sstream << str;

# 字符流转string
const string& str2 = ss.str();

# string转int(转任意类型)
stringstream steam;
int a = 0;
steam << str;
steam >> a;
```

## 字符串视图

* C++17支持
* 仅能用于字符串查找、遍历（只读），而不能修改

# 格式化

* C++20支持
* 具体选项参考指定格式数据

```c++
std::string s = fmt::format("I'd rather be {1} than {0}.", "right", "happy");
std::string s = fmt::format("{}, {}, {}", 'a', 'b', 'c');

#include <stdio.h>
char temp[14];
sprintf(temp,"hello world%d\n",i);
# 限制输出的字符数，避免缓冲区溢出
int j = snprintf(buffer, sizeof(buffer)-1, "This is %s\n", s);
```

# 子串

```C++
s[i,j] == s.substr(i,j-i+1)
s[i,j] == s.substr(i,j-i)
s[i,.. == s.substr(i)
```

# 替换

```C++
# 替换指定位置
# 从第一个a位置开始的两个字符替换成
str=str.replace(str.find("a"),2,"#");
# 用#替换从begin位置开始的5个字符
str=str.replace(str.begin(),str.begin()+5,"#");

# 替换指定字符
# 每次从头开始替换
string& replace_all(string& str, const string& old_value, const string& new_value)
{
	while( true ) {
		string::size_type pos(0); //  string::size_type rc = s.find(.....)
		if( (pos=str.find(old_value)) != string::npos )  //注意pos = str.find(...)要加括号
			str.replace(pos,old_value.length(),new_value);
		else 
			break ;
	}
	return str;
}
# 每次从上一次替换的位置开始
string& replace_all_distinct(string& str, const string& old_value, const string& new_value)
{
	for(string::size_type pos(0); pos!=string::npos; pos+=new_value.length()) {
		if( (pos=str.find(old_value,pos))!=string::npos )
			str.replace(pos,old_value.length(),new_value);
		else 
			break ;
	}
	return	str;
}
```

# 复制

* 使用内存复制
* strcpy 复制整个字符串

  ```C++
  # 返回char* 链式调用,如int length = strlen( strcpy( strDest, “hello world”) );
  char *strcpy(char *strDest, const char *strSrc)
  {
      assert((strDest!=NULL) && (strSrc !=NULL));
      char *address = strDest;
      while( (*strDest++ = * strSrc++) != '\0' )
      return address ;
  }
  ```
* strncpy 复制字符串的前n个字符

  ```C++
  char *strncpy(char *dest, const char *src，int n);
  strncpy(dest, src, 10);
  char* strncpy(char* dest, const char* src,size_t count)
  {
  	char* start = dest; // 记录目标字符串起始位置
  	while (count && (*dest++ = *src++)) // 拷贝字符串
  	{
  		count--;
  	}
  	if (count) // 当count大于src的长度时，将补充空字符
  	{
  		while (--count)
  		{
  			*dest++ = '\0';
  		}
  	}
  	return start;
  }
  ```
* strdup 复制整个字符串（自动分配空间），使用完后需要free内存

  ```C++
  char *strdup(const char *s);
  char *dest = strdup(s);
  static char *strdup(char *s)
  {
      char * t;
      if (!s)
          return NULL;
      t = (char *)malloc(strlen(s) + 1);
      if (t) {
          strcpy(t, s);
      }
      return t;
  }
  ```

# 删除

删除子串（包括删除后重新连接的子串）可以使用栈：[2696. 删除子串后的字符串最小长度 - 力扣（LeetCode）](https://leetcode.cn/problems/minimum-string-length-after-removing-substrings/submissions/494438513/)

```C++
# 从第6个位置（默认为0）开始，删除4个字符
str.erase(6, 4);

# 删除最后一个字符
void pop_back();

# 删除某个字符
iterator erase (iterator p);

# 删除某个区间[first,last)
iterator erase (iterator first, iterator last);
```

# 实现

```C++
#include <cstring>
#include <iostream>
using namespace std;

class String
{
public:
    String(const char *str = NULL);       // 普通构造函数
    String(const String &str);            // 拷贝构造函数
    String &operator=(const String &str); // 赋值函数
    ~String();                            // 析构函数
    const char *c_str() const;

private:
    char *m_data; // 用于保存字符串
};

const char *String::c_str() const
{
    return m_data;
}
// 普通构造函数
String::String(const char *str)
{
    if (str == nullptr)
    {
        m_data = new (std::nothrow) char[1]; // 对空字符串自动申请存放结束标志'\0'的空间
        if (m_data == nullptr)
        { // 内存是否申请成功
            std::cout << "申请内存失败！" << std::endl;
            exit(1);
        }
        m_data[0] = '\0';
    }
    else
    {
        int length = strlen(str);
        m_data = new (std::nothrow) char[length + 1];
        if (m_data == nullptr)
        { // 内存是否申请成功
            std::cout << "申请内存失败！" << std::endl;
            exit(1);
        }
        strcpy(m_data, str);
    }
}

// 拷贝构造函数
String::String(const String &other)
{ // 输入参数为const型
    int length = strlen(other.m_data);
    m_data = new (std::nothrow) char[length + 1];
    if (m_data == nullptr)
    { // 内存是否申请成功
        std::cout << "申请内存失败！" << std::endl;
        exit(1);
    }
    strcpy(m_data, other.m_data);
}

// 赋值函数
String &String::operator=(const String &other)
{                       // 输入参数为const型
    if (this == &other) // 检查自赋值
    {
        return *this;
    }

    delete[] m_data; // 释放原来的内存资源

    int length = strlen(other.m_data);
    m_data = new (std::nothrow) char[length + 1];
    if (m_data == nullptr)
    { // 内存是否申请成功
        std::cout << "申请内存失败！" << std::endl;
        exit(1);
    }
    strcpy(m_data, other.m_data);

    return *this; // 返回本对象的引用
}

// 析构函数
String::~String()
{
    delete[] m_data;
}

int main()
{
    String a;
    String b("abc");
    a = b;
    cout << a.c_str() << endl;
    system("pause");
}
```
