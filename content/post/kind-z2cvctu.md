---
title: 类
slug: kind-z2cvctu
url: /post/kind-z2cvctu.html
date: '2024-07-10 20:24:49+08:00'
lastmod: '2024-08-08 13:04:20+08:00'
toc: true
tags:
  - 类
categories:
  - 语法
keywords: 类
isCJKLanguage: true
---

# 类

# struct

||class|struct|
| --------------------------------| ---------| --------|
|默认访问权限|private|public|
|默认继承|private|public|
|定义模板参数<br />（代替typename）|√|×|

>  C++对struct进行了拓展，和C中的struct不同

# 封装

与C中的struct不同，类将数据和函数结合起来，使用访问权限来隐藏实现细节，提供对外接口

## 访问控制

||子类|友元|类外|
| -----------| ------| ------| ------|
|public|√|√|√|
|protected|√|√|×|
|private|×|√|×|

# 继承

内存布局<sup>（https://www.cnblogs.com/QG-whz/p/4909359.html）</sup>：虚函数表指针vfptr <sup>(如果父类没有虚函数，则没有)</sup>+ 基类非静态成员 + 派生类非静态成员

* 默认private继承

## 多继承

内存布局：（虚函数表指针vfptr<sup>(如果基类中没有虚函数表，则不需要vfptr)</sup> + 基类非静态成员<sup>（如果基类存在基类，则先写基类的基类的成员变量，再写基类的成员变量）</sup>） * 基类个数<sup>（按继承顺序）</sup> + 派生类非静态成员

使用：

```c++
struct IMachine: IPrinter, IScanner
class Derived: public BaseA, public BaseB { }
```

## 虚拟继承

内存布局：（虚函数表指针vfptr + 虚基类指针vbptr + 基类非静态成员<sup>（不包含基类的基类）</sup>）* 基类个数 + 虚祖父类非静态成员<sup>（如果仅是B虚继承A,则没有虚祖父类；仅在虚拟菱形继承中存在）</sup> + 派生类非静态成员

原理：原本的父类成员会被替换为 一个**虚基类指针vbptr**，这个指针指向一张**虚基类表**，虚基类表里存放 **vbptr到该类首地址的偏移**。

作用：

1. 解决多继承/菱形继承的“数据冗余”：在一个派生类中保留间接基类的多份同名成员，例如A派生B和C，D继承了B和C，则A的成员变量和成员函数在D中有两份。**虚继承中仅保留一份间接基类的成员**。
2. 解决命名冲突：D如果想访问A的成员变量，则编译器不知道它究竟来自 A -->B-->D 这条路径，还是来自 A-->C-->D 这条路径，产生二义性，需要使用B::m_a来指明。

使用：

```c++
class B: virtual public A{}
```

# 多态

含义：根据不同的对象类型，调用不同的函数

## 编译时多态（静态多态）

**实现**：

* 模板template
* 函数重载
* CRTP

**原理**：在程序编译阶段，编译器通过**实参匹配形参**的方式确定具体调用哪个函数

## 运行时多态（动态多类）

**实现**：

1. 继承
2. 父类 **指针<sup>（父类对象不行;指针或引用只要求基地址和内存大小,不引发内存任何“与类型有关的内存委托操作;派生类对象直接赋值给基类对象，就牵扯到对象的类型问题）</sup>**​**或引<sup>（父类对象不行;指针或引用只要求基地址和内存大小,不引发内存任何&quot;与类型有关的内存委托操作&quot;;派生类对象直接赋值给基类对象，就牵扯到对象的类型问题）</sup>**​**用<sup>（父类对象不行;指针或引用只要求基地址和内存大小,不引发内存任何“与类型有关的内存委托操作;派生类对象直接赋值给基类对象，就牵扯到对象的类型问题）</sup>** 指向子类对象
3. 父类虚函数 + 子类重写父类的虚函数

原理：虚函数表

# 成员变量

1. 访问权限控制：如果基类中的成员变量需要被子类访问，设为protected；其余设置为private
2. 初始化：在定义时使用 = 或 {} 初始化

    C++11前仅能在构造函数的初始化列表中赋值

    C++11后可以在定义时赋值

    ```c++
    class Foo(){
    private:
        vector<string> name = vector<string>(5);
        vector<int> val{vector<int>(5,0)};
    }
    ```

3. 指针或对象：

    对象：无需进行内存管理，无需担心空指针

    指针：

    * 当成员变量要使用多态时
    * 当成员变量是可选时，有时不会使用，节省空间
    * 当成员变量很大时
    * 当成员变量对应的资源不是对象独有，而是多个对象共同所有

4. 引用

    * 构造函数的形参也必须是引用类型
    * 不能在构造函数里初始化，必须在**构造函数的初始化列表**中初始化
5. const 只读成员变量

    * 只能在**构造函数的初始化列表**中初始化

## 静态成员变量

必须在类外<sup>（不能在类中、初始化列表中初始化）</sup>初始化

作用：在所有实例之间共享数据；将数据存储在类中

```C++
class class_name{
private:
	static int x;
};
int class_name::x = 0;
```

# 成员函数

## 虚函数

作用：实现多态；重写函数<sup>（如类库、框架源码不能修改时使用虚函数重写）</sup>

纯虚函数：定义接口，不需要实现，抽象类不能实例化，**仅能定义指针和引用**，使用多态

**特点**：

* 可以是内联函数，通过对象调用虚函数可以内联，但虚函数表现多态<sup>（使用基类指针调用子类对象）</sup>时会忽略inline
* 可以是private，且可被子类覆盖；但需要检查使用多态指针的函数<sup>（如main函数，并不是指虚函数）</sup>必须为友元
* 如果对象没有实例化<sup>（调用构造函数之后才有虚函数表指针）</sup>，不能调用虚函数
* **override** 编译器会检查父类是否有同名方法，且更易阅读
* 不能是成员模板函数template

  函数模板在**声明**时编译一次，在**调用**时编译一次。只有当所有编译单元完成时，才能确定被展开为多少种形式

  虚函数表在**类编译完成时**已经被确定

实现：每个**对象**有虚函数表指针 + 每个**类**有虚函数表（父类和子类有各自的虚函数表）

* 子类可以继承<sup>（子类不重写）</sup>、覆盖<sup>（子类重写）</sup>、添加虚函数表中的函数
* 父类和子类不共用虚函数表，但表中虚函数地址可能相同（子类没有重写）。
* 多态时，父类指针查询的是**子类的虚函数表**，所以可以调用子类的重写函数。
* 虚函数表是类编译期间建立的

```c++
# 虚函数
virtual void draw(){}

# 纯虚函数
virtual void draw() = 0;

# 子类重写函数（同函数名、同参数列表、同返回类型）
void draw() override {
	cout << "Drawing a circle" << endl;
}
```

## 运算符

* 不能定义新的运算符
* 不能重载的运算符：成员运算符<sup>（A.a）</sup>、指针运算符<sup>（p->a）</sup>、作用域运算符<sup>（A::func()）</sup>、sizeof、三元条件运算符<sup>（?:）</sup>
* 只能使用成员函数重载的运算符：=、()、[]、->、new、delete
* 重载运算符的限制

  1. 不能改变运算符**操作数的个数**
  2. 不能改变运算符原有的**优先级**
  3. 不能改变运算符原有的**结合性**
  4. 不能改变运算符原有的**语法结构**

```c++
# 全局函数
complex operator+(const complex& c1, const complex& c2);

# 成员函数(自带一个this指针参数,所以二元函数仅需一个参数)
complex complex::operator+(const complex& c);

# 友元函数(没有this指针参数)
friend ostream& operator<<(ostream &os,const complex &c)

# ++x(前置++)
student& operator++(void){
	++this->num;
	return *this;
}

# x++(后置++,使用int为参数)
student operator++(int){
	class student temp(*this);
	++this->num;
	return temp;
}
```

## 判断类是否有某函数SFINAE

SINAE：Substitution Failure Is Not An Error 匹配失败不是错误

只能判断是否有对应的public函数（看不到private函数）

参考类型萃取

```C++
# 不能看到继承的成员函数
template<typename T>
struct has_destroy
{
  template <typename C> static char test(decltype(&C::destroy));
  template <typename C> static int32_t test(...);
  const static bool value = sizeof(test<T>(0)) == 1;
};
cout<< has_destroy<X>::value <<endl;
```

```C++
# C++11后，编译时只能-std=c++11
# 原理参考类型萃取
#include <iostream>
#include <list>
#include <map>
#include <set>
#include <string>
#include <vector>
#include <utility> // std::declval
template <typename>
using void_t = void;
// #include <type_traits>

template <typename T, typename V = void>
struct has_push_back:std::false_type {};

template <typename T>
struct has_push_back<T, void_t<decltype(std::declval<T>().push_back(std::declval<typename T::value_type>()))>>:std::true_type {};

int main() {
    std::cout << has_push_back<std::list<int>>::value << std::endl;
    std::cout << has_push_back<std::map<int, int>>::value << std::endl;
    std::cout << has_push_back<std::set<int>>::value << std::endl;
    std::cout << has_push_back<std::string>::value << std::endl;
    std::cout << has_push_back<std::vector<int>>::value << std::endl;
    return 0;
}
----------------------------------------------------------------------------
template <typename>
using void_t = void;

template <typename T, typename V = void>
struct has_ss_type : std::false_type {};

template <typename T>
struct has_ss_type <T, void_t<decltype(std::declval<std::stringstream>().operator<<(std::declval<T>()))>> : std::true_type {};

int main() {
    std::cout << has_ss_type<int>::value << std::endl;
    std::cout << has_ss_type<double>::value << std::endl;
    std::cout << has_ss_type<std::string>::value << std::endl;
    std::cout << has_ss_type<std::vector<int>>::value << std::endl;
    return 0;
}
```

```C++
# C++11前
# 两个Helper表示push_back的参数类型有两种：value_type和const value_type&
struct has_push_back {
	// value_type：容器存储的类型
    template <typename C, void (C::*)(const typename C::value_type&)>
    struct Helper;

    template <typename C, void (C::*)(typename C::value_type)>
    struct Helper2;

    template <typename C>
    static bool test(...) {
        return false;
    }
    template <typename C>
    static bool test(Helper<C, &C::push_back>*) {
        return true;
    }
    template <typename C>
    static bool test(Helper2<C, &C::push_back>*) {
        return true;
    }
};

int main() {
    std::cout << has_push_back::test<std::list<int> >(NULL) << std::endl;
    std::cout << has_push_back::test<std::map<int, int> >(NULL) << std::endl;
    std::cout << has_push_back::test<std::set<int> >(NULL) << std::endl;
    std::cout << has_push_back::test<std::string>(NULL) << std::endl;
    std::cout << has_push_back::test<std::vector<int> >(NULL) << std::endl;
    return 0;
}
```

```C++
# Boost
#include <boost/tti/has_member_function.hpp>
#include <string>
#include <iostream>

struct ClassWithToString {
    std::string to_string() {   return "with to_string"; }
};
struct ClassWithToStringConst {
    std::string to_string() const {   return "with to_string"; }
};
struct ClassWithoutToString {
};

BOOST_TTI_HAS_MEMBER_FUNCTION(to_string);

int main() {
    std::cout << std::boolalpha
        << has_member_function_to_string<ClassWithToString, std::string>::value << std::endl
        // true
        << has_member_function_to_string<ClassWithToStringConst, std::string>::value << std::endl
        // false
        << has_member_function_to_string<const ClassWithToStringConst, std::string>::value << std::endl
        // true
        << has_member_function_to_string<ClassWithoutToString, std::string>::value << std::endl;
        // false
}
```

## 静态成员函数

不能访问**非静态成员变量** 和 **非静态成员函数**，无this指针参数

作用：

* **构造函数或析构函数**可以调用静态成员函数；
* 静态成员函数可以调用构造函数

```C++
class class_name{
public:
	static void func(){
		...
	}
};
class_name::func();
```

## CRTP（调用子类函数）

本质：继承了不同的基类

作用：父类**实现时**调用子类函数、父类函数返回子类对象

```C++
template <typename Derived>
struct SomeBase
{
	void foo(){
		for(auto &iterm : *static_cast<Derived *>(this)){...}
	}
}

# 实现链式引用
template <typename Derived>
class Printer
{
public:
  Printer(std::ostream& pstream): m_stream(pstream){}

  template<typename T>
  Derived& print(T&& t)
  {
    m_stream << t;
    return static_cast<Derived&>(*this);
  }
private:
  std::ostream& m_stream;
};

//Derived class
class CoutPrinter : public Printer<CoutPrinter>
{
public:
    CoutPrinter() : Printer(std::cout) {}

    CoutPrinter& SetConsoleColor(Color c)
    {
        switch (c)
        {
        case Color::red:
          std::cout << "\033[0;31m " << std::endl;
          break;
        default:
          break;
        }
        return *this;
    }
};

int main(int argc, char* argv[])
{
  CoutPrinter().print("Hello ").SetConsoleColor(Color::red).println("Printer!");
}
```

## 默认成员函数

如果存在自定义构造函数，则不会生成无参构造函数，可通过`=default`​来让编译器生成无参构造函数，`=delete`​禁止生成指定函数

* 构造函数
* 析构函数
* 拷贝构造函数
* 拷贝赋值函数
* 取地址重载
* cosnt取地址重载
* 移动构造函数
* 移动赋值函数

# 大小sizeof

1. 一个对象的大小大于等于所有**非**​**静态成员<sup>（静态（static）成员变量不在对象中存储）</sup>**大小的总和。
2. **空类**的大小为1个字节，当类不包含虚函数和非静态数据成员时，其对象大小也为1。

    > 因为空类也可以实例化很多对象，需要实现每个实例在内存中都有一个独一无二的地址。
    >

3. 虚函数：增加指针<sup>（32位机器为4字节，64位机器为8字节）</sup>指向虚函数<sup>（虚函数本身不占用空间）</sup>表VTable
4. 字节对齐：注意包括虚函数表指针（8字节或4字节）
5. 单继承：参考内存布局，注意每个字段都需要字节对齐
6. 多继承：参考内存布局，注意每个字段都需要字节对齐

    > 注意菱形继承，可能存在多份基类数据
    >

7. 虚继承：参考内存布局，注意每个字段都需要字节对齐

```c++
#include<iostream>
using namespace std;

class A {
    int a;
    virtual void myfuncA(){}
};
  
class B:virtual public A{
    virtual void myfunB(){}
};
 
class C:virtual public A{
    virtual void myfunC(){}
};
 
class D:public B,public C{
    virtual void myfunD(){}
};
 
int main(void) 
{ 
    cout<<"A="<<sizeof(A)<<endl;    //result=16 
    cout<<"B="<<sizeof(B)<<endl;    //result=24   
    cout<<"C="<<sizeof(C)<<endl;    //result=24 
    cout<<"D="<<sizeof(D)<<endl;    //result=32
    return 0; 
}
```

```c++
#include<iostream>
using namespace std;

class A  
{  
};   
 
class B  
{ 
    char ch;  
    virtual void func0()  {  }  
};  
 
class C   
{ 
    char ch1; 
    char ch2; 
    virtual void func()  {  }   
    virtual void func1()  {  }  
}; 
 
class D: public A, public C 
{  
    int d;  
    virtual void func()  {  }  
    virtual void func1()  {  } 
};  
class E: public B, public C 
{  
    int e;  
    virtual void func0()  {  }  
    virtual void func1()  {  } 
}; 
 
int main(void) 
{ 
    cout<<"A="<<sizeof(A)<<endl;    //result=1 
    cout<<"B="<<sizeof(B)<<endl;    //result=16   
    cout<<"C="<<sizeof(C)<<endl;    //result=16 
    cout<<"D="<<sizeof(D)<<endl;    //result=16 
    cout<<"E="<<sizeof(E)<<endl;    //result=32 
    return 0; 
}
```

```C++
struct MyStruct {
    int a;
    double b;
    char c;
};
sizeof(MyStruct) = 24
```

## 字节对齐

作用：减少**CPU访问内存**数据的次数，提高取数据的效率

1. 结构体变量的**起始地址**能够被其最宽的成员大小整除
2. 结构体每个成员相对于起始地址的偏移**能够被其自身大小整除**，如果不能则在前一个成员后面补充字节
3. 结构体总体大小能够被**最宽的成员的大小**整除，如不能则在后面补充字节

原理：以向上对齐为例

1. 将len加上ALIGNVAL-1
2. 将结果与 ~(ALIGNVAL-1)<sup>(将最后的n位转为0)</sup> 进行按位与，确保是**ALIGNVAL的倍数**

```C++
# 向上对齐，ALIGNVAL对齐大小必须为2的n次幂
#define TYPEALIGN(ALIGNVAL,LEN) \
	(((uintptr_t) (LEN) + ((ALIGNVAL) - 1)) & ~((uintptr_t) ((ALIGNVAL) - 1)))

# 向下对齐，ALIGNVAL对齐大小必须为2的n次幂，如地址所属页的起始地址
#define TYPEALIGN_DOWN(ALIGNVAL, LEN) \
	(((uintptr_t)(LEN)) & ~((uintptr_t)((ALIGNVAL)-1)))
```

# 构造函数

* 无返回类型
* 尽量不要抛出异常：无法调用析构函数，可能导致内存泄漏
* 可以函数重载
* **不能是虚函数**

  * 虚函数需要通过对象实例化后的虚函数表来实现，即虚函数表在调用构造函数后才建立。
  * 虚函数是允许基类指针可以调用子类的方法，而构造函数是子类自己调用的，不可能通过父类的指针或引用去调用
* 调用顺序：基类 - 嵌套类 - 自身

```c++
DropTableStmt(const std::string &table_name)
        : table_name_(table_name), next(nullptr)   // 初始化列表
{}
```

## 删除构造函数

* 不允许无参构造

  ```c++
  class_name()=delete;
  ```

* 不允许类外构造实例，例如单例模式

  ```c++
  private:
  	class_name(){ }
  ```

## 禁止隐式类型转换

作用：只能显式调用构造函数，防止意外转换

场景：**单参数**的构造函数

```c++
explicit Entity(){}
```

## 子类构造函数

子类创建时会**首先调用父类（虚继承优先）的无参构造函数**，按`声明顺序`​初始化成员变量，然后调动自身的构造函数

如果**父类只有有参构造函数**，则子类需要显式调用父类的构造函数

```c++
son():father(3,2) { a=3; }
```

# 拷贝构造函数

作用：对指针和引用类型进行深拷贝<sup>（分配内存和赋值）</sup>

```c++
class_name(const class_name& other){
	ptr = new ...;
	data_ = othrer.data_;  // 非指针成员进行浅拷贝
}
```

场景：

1. 初始化同类对象（该对象未被初始化）

    ```c++
    Complex c2(c1);
    Complex c2 = c1;
    ```

2. 按值传递参数

## 禁止复制

1. 使用delete；也可设置为private，同时不提供实现

```c++
class_name(const class_name&) = delete;
Class_name& operator=(const Class_name &) = delete;
```

2. 宏定义

```shell
#define DISALLOW_COPY( cname )                    \
	cname(const cname &) = delete; /* NOLINT */    \
	auto operator=(const cname &)->cname & = delete;  /* NOLINT */

#define DISALLOW_MOVE( cname )                   \
	cname( cname && ) = delete; /* NOLINT */      \
	auto operator=(cname &&)->cname & = delete;  /* NOLINT */

#define DISALLOW_COPY_AND_MOVE( cname )       \
	DISALLOW_COPY(cname);                         \
	DISALLOW_MOVE(cname);
```

3. 继承noncopyable类

```C++
class noncopyable
{
public:
	noncopyable(const noncopyable&) = delete;
	void operator=(const noncopyable&) = delete;

protected:
	noncopyable() = default;
	~noncopyable() = default;
};
class BlockingQueue : noncopyable {}
// 默认private继承
```

## 可复制

```C++
class copyable
{
 protected:
  copyable() = default;
  ~copyable() = default;
};
class BlockingQueue : copyable {}
```

# 移动构造函数

作用：以右值临时对象初始化同类对象（未初始化）

```c++
class_name(class_name && other): object(std::move(other.object)) {
	this->data = other._data; //将源对象的数据成员的值赋给目标对象
	other._data = nullptr; //将原来内容置空
}
# 使用
class_name new_owner(std::move(old_owner))
```

## 禁止移动

参考禁止复制

```c++
class_name(classname &&) = delete;  //移动构造
class_name & operator=(class_name &&) = delete;  //移动赋值
```

# 转换构造函数

作用：将其它类型转换为当前类类型

特点：

* 只能有一个参数的构造函数

```c++
string(char *);
Complex(double real): m_real(real), m_imag(0.0){ }
```

# 析构函数~

* 不能抛出异常：

  * 导致后续的对象无法被析构，从而资源泄漏
  * C++11默认析构函数是noexcept的
  * 很难被捕获（栈对象离开栈时才调用析构函数，如果在try{}中无法被捕获）
* 不能被重载：构造函数只能有一个，且不能带参数
* delete时**自动调用**析构函数，并free内存
* 基类的析构函数必须为**虚函数**

  使用多态时，如果父类析构函数没有virtual，则不会调用子类的析构函数
* 调用顺序：自身 - 嵌套类 - 父类

作用：释放内存

```c++
virtual ~class_name() { }  // 基类需要使用virtual
e.~class_name();   // 手动调用析构函数
```

## 子类析构函数

析构顺序和构造顺序完全相反

* 栈上对象：先调用子类析构函数，再调用父类析构函数
* 堆上对象：

  **父类指针指向子类对象**：如果父类析构函数没有virtual，则不会调用子类的析构函数（如果有virtual，则会调用）

  **子类指针指向子类对象**：无论是否有virtual，则都会调用父类和子类的析构函数

# 赋值函数

场景：将对象赋值给已初始化的对象

> 实现中内存复制可参考内存复制

```C++
# 赋值函数
Class_name& operator=(const Class_name &other){
	if(this!= &other){
		this->_len = other->_len; // 浅拷贝

		if (_date)  //删除原来的内存
     		delete[] this->_data;
		this->_data = (int*)calloc(this->_len,sizeof(int)); // 分配内存
		memcpy(this->_data,other->_data,_len*sizeof(int)); // 内存复制
		other._data = nullptr; // 将原指针置空

		this->object = std::move(that.object); //可移动对象不需要上面三个步骤
	}
	return *this;
}
# 移动赋值函数
class_name & operator=(class_name && other);
# 转换赋值函数
unique_ptr& operator=(T* p);
// 参数可以为其他类型
```

## 禁用赋值

参考对应的禁止复制、禁止移动

# 位域

作用：压缩内存

```C++
struct bs 
{ 
    int a:8; 
    int b:2; 
    int c:6; 
}data;
// data为bs变量，其中位域a占8位，位域b占2位，位域c占6位(一个字节8位)
struct bs
{
	unsigned a:4
	unsigned  :0 /*空域*/
	unsigned b:4 /*从下一单元开始存放*/
	unsigned c:4
};
// a占第一字节的4位，后4位填0表示不使用，b从第二字节开始，占用4位，c占用4位

data.a = 0;
```
