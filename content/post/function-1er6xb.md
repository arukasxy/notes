---
title: 函数
slug: function-1er6xb
url: /post/function-1er6xb.html
date: '2024-07-10 23:45:33+08:00'
lastmod: '2024-08-08 13:25:49+08:00'
toc: true
tags:
  - 函数
categories:
  - 语法
keywords: 函数
isCJKLanguage: true
---

# 函数

# 函数重载 函数重写 函数隐藏

|函数重载|函数重写（覆盖）|函数隐藏|
| :-----------------------------------------: | :----------------------------------: | ------------------------------|
|相同的函数名<br />|||
|不同的参数列表<sup>（函数的参数数目和类型，以及参数的排列顺序 不同也可）</sup>|相同的参数列表|---|
|返回值类型可以不同|相同的返回类型|---|
|相同作用域：同一个类中/全局函数|父类和子类：virtual 虚函数|父类和子类|
|编译期间：通过形参的类型确定|运行期间：通过调用者的实际类型确定|子类对象不能调用父类同名函数|
|使函数可以处理多种输入参数|多态||
|参考下面、函数模板|参考虚函数||

```C++
# 函数重载
void print(const char* str,int width);
void print(double i ,int width);
void print(const char* str);
```

# 可调用对象（函数对象）

## 普通函数

```c++
bool cmp(const int &a, const int &b) {
	return a < b; // 从小到大排列
   //return a > b; // 从大到小排列
}
sort(v.begin(), v.end(), cmp);
```

## 函数指针

函数返回值类型 (* 指针变量名) (函数参数列表);

```c++
bool cmp(const int &a, const int &b) {
	return a < b; // 从小到大排列
    //return a > b; // 从大到小排列
}
bool (*p)(const int &a, const int &b); // p为函数指针
p = cmp; // 赋值

sort(v.begin(), v.end(), p); # 作为传入参数
bool tmp = (*cmp)(3, 4); # 调用(*函数指针)(调用参数)
```

## 仿函数

特点：

* 重载operator()运算符的类对象
* 不能传给接收函数指针的参数
* 可以存储之前的调用结果，不使用全局静态变量

```c++
class Bigger {
private:
	int base_;
public:
	Bigger(){}
	Bigger(int base) {
		base_ = base;
	}
	void operator() (const int &num) {
		cout << (num > base_);
	}
};

Bigger big(base);  
cout<< Bigger(base)(5);  # 使用临时对象
cout<< big.operator()(5);  # 通过对象显式调用
cout<< big(5); # 通过对象隐式调用

# 作为传入参数
for_each(v.begin(), v.end(), big);
for_each(v.begin(), v.end(), Big());
```

## 匿名函数（lambda）

* 在lambda中改变函数外的变量需要加`mutable`​

```c++
#include <functional>
int tmp = 0;
# 也可以使用auto来代替
function<int(int)> mylam = [&tmp](int value) mutable throw() -> int
{
	tmp = 1;
	std::cout << "value:" << value <<std::endl;
	return tmp;
};

for_each(v.begin(), v.end(), [base](const int &num){cout << (num > base);});
--------------------
auto cmp = [](const pair<string, int>& a, const pair<string, int>& b) {
	return a.second == b.second ? a.first < b.first : a.second > b.second;
}; 
priority_queue<pair<string, int>, vector<pair<string, int>>, decltype(cmp)> que(cmp);
set<pair<string,int>, decltype(cmp)> pq;
--------------------
# 在类中定义STL时，使用function封装，避免声明时cmp不存在，及cmp中需要this指针
```

|参数|含义|
| ---------| ------------------------------------------------------------------------------------------|
|[]|不捕获任何外部变量|
|[=]|通过 **值传递** 捕获外部作用域中 所有变量|
|[&]|通过 **引用传递** 捕获外部作用域中 所有变量|
|[a]|通过 **值传递** 捕获a变量|
|[&a]|通过 **引用传递** 捕获a变量|
|[this]|捕获当前类中的this指针，让lambda表达式拥有和当前类成员函数同样的访问权限<br />在表达式中使用(void) this;<sup>(表示不使用this指针)</sup>|
|[=,&a]|按值捕获所有变量，按引用捕获a变量|
|mutable|允许函数体修改通过值传递捕获的参数<sup>（不会将修改传递到lambda表达式外）</sup>|

### 递归调用自己

* 使用function为其赋予变量 + [&]引用传递

  ```C++
  std::function<int(const int&)> s = [&](const int& n) {
  	return n == 1 ? 1 : n + s(n - 1);
  };
  ```
* 将lambda作为参数

  ```C++
  const auto& s = [&](auto&& self, const int& x) -> int{
  	return x == 1 ? 1 : x + self(self, x - 1);
  };
  ```

## std::function

特点：

* 可兼容所有C++的可调用对象
* 类成员函数需要使用bind绑定this指针后才可作为function

```c++
#include <functional>
std::function<int(int, int)> Func;

int res = Func(1,2);

# 在类中定义STL时，使用此方法，避免声明时cmp不存在，及cmp中需要this指针
set<pair<int,int>,function<bool(const pair<int,int>&, const pair<int,int>&)>> pq;
function<bool(const pair<int,int>&, const pair<int,int>&)> cmp = 
	[this](const pair<int,int> &a, const pair<int,int> &b){
    	return distance(a)==distance(b)?b.first > a.first:distance(a)>distance(b); };
pq = set<pair<int,int>,function<bool(const pair<int,int>&, const pair<int,int>&)>>(cmp);
```

### bind 函数参数绑定

特点：

* 不预绑定的参数使用 _1 等占位符
* 引用变量需要使用 ref、cref
* 类外bind成员函数需要传入 引用或指针

  类中bind成员函数需要传入 this指针

```c++
#include <functional>
auto f = bind(g, 1, 2); // = g(1,2)
auto f = bind(g,_1, 2); // _1为不预绑定的参数
auto f = bind(g, ref(x), _1); // = g(x)
auto f = bind(&X::f, args); // 成员函数

X x;
shared_ptr<X> p(new X);
auto f = bind(&X::f, ref(x), _1)(i);
auto f = bind(&X::f, &x, _1)(i);
auto f = bind(&X::f, p, _1)(i);
auto f = bind(&X::f, this); // 类内使用this
```

### 回调函数

使用bind绑定参数后即可适配std::function<void ()>

```C++
typedef std::function<void ()> Task;
void func(int &x);
int i=0;
Task task = std::bind(func,std::ref(i));
std::thread t(task);
```

## 成员函数

```c++
# 传入类对象地址
std::call_once(initDataFlag, &X::func, this);

# 使用mem_fn封装
Foo f;
auto func = std::mem_fn(&Foo::display);

func(f);
std::for_each(threads.begin(),threads.end(),std::mem_fn(&std::thread::join));
```

## STL自带仿函数（算术）

```c++
#include <functional>
negate<int>n;
cout << n(10) << endl; //-10
sort(v.begin(), v.end(), greater<int>());
```

|函数|功能|
| ---------------------------------| -------|
|plus\<T\>|+|
|minus\<T\>|-|
|multiplies\<T\>|*|
|divides\<T\>|/|
|modulus\<T\>|%取模|
|negate\<T\>|取反|
|equal\_to\<T\>|==|
|not\_equal\_to\<T\>|!=|
|greater\<T\>|>|
|greater\_equal\<T\>|>=|
|less\<T\>|<|
|less\_equal\<T\>|<=|

# 修饰词

|修饰词|含义|场景|
| -----------| ------------| --------------------------------|
|constexpr|编译时求值|返回类型和所有形参的类型必须是**字面值类型**|
|inline|内联函数<sup>（在函数的调用处展开）</sup>|函数体非常短小且会被频繁调用|

## 内联函数

* 编译时展开
* 可以进行类型安全检查等
* 编译器可以决定是否内联

# 运算符

参考运算符

# 可重入函数

可以被**中断**的函数

可重入函数**一定线程安全**，线程安全函数不一定可重入

需满足以下条件：

1. 可以在执行的过程中可以被打断
2. 被打断之后，在该函数一次调用执行完之前，可以再次被调用
3. 再次调用执行完之后，被打断的上次调用可以继续恢复执行，并正确执行

通常需要满足：

* 不能含有静态或全局非常量数据<sup>（可重入的全局数据必须只读）</sup>。
* 不能返回静态或全局非常量数据的地址
* 不能调用标准 I/O
* 调用的函数也必须是可重入的
* 没有动态分配或释放堆资源

# 链式调用

如果涉及类的继承，参考CRTP

```C++
# 与类的继承无关，返回指针或引用
strcpy char *strcpy(char *strDest, const char *strSrc); // strcpy能把strSrc的内容复制到strDest，为什么还要char * 类型的返回值?
int length = strlen( strcpy( strDest, “hello world”) );

complex& func(complex *ths){
	return *ths;
}
```

# 传入参数

|方式|作用|
| ----------------------| --------------------------------------------------|
|传值||
|枚举|可选性，代替`bool`​|
|数组T[ ]|**退化为指针**，需传入数组大小n，或传入数组引用 T (&arr)[10]|
|T&|需修改参数|
|const T&|保存参数，常用于类中构建拷贝构造函数|
|T&&|保存参数，调用时使用std::move、完美转发|
|unique_ptr<T>|转移所有权，调用时使用std::move( unique_ptr<T> )|
|unique_ptr<T>&|改变管理的资源|
|shared_ptr<T>|资源的共享所有者|
|shared_ptr<T>&|改变管理的资源，不更新引用计数|
|const shared_ptr<T>&|最外层函数中持有shared_ptr<T>，内层函数使用`const shared_ptr<T>&`​|

使用智能指针的引用，不延长资源的生命周期

## 可变参数

可配合tuple使用

```C++
# 递归调用，需设置递归终止函数
template<typename T, typename... Types>
T sum(T first, Types... rest) {
    return first + sum<T>(rest...);
}

template<typename T>
T sum(T t) {
    return t;
}

std::tuple<int,int,int> t(1, 2, 3);
cout<< std::apply([](auto &&... args) {return sum(args...);},std::move(t))<<endl;
```

# 返回类型

* 函数不能返回局部变量/对象的引用或指针，但可以**返回局部变量/对象**

  根据返回值优化RVO技术，可能通过拷贝或移动构造临时变量

|方式||
| ----------------------------------------------| ----------------------------------|
|对象||
|bool<br />指针<br />std::optional<br />|返回函数是否执行成功|
|引用<br />指针|链式调用<br />不需要使用引用形式接收|
|数组(相同类型)<br />元组(不同类型)<br />结构体struct|多返回值|

# 函数调用过程

1. 输入参数压栈
2. 返回地址压栈，调用函数
3. ebp压栈，保存上一栈帧的栈底<sup>（上一栈帧的栈顶esp就是本函数的栈底）</sup>
4. ebp设置为esp，上一栈帧的栈顶esp就是函数的栈底
5. 为函数局部变量分配栈空间
6. 执行完成后，ebp弹栈
7. 销毁栈帧，恢复寄存器的旧值

‍
