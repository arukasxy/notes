---
title: 内存管理
slug: memory-management-z1c7hkf
url: /post/memory-management-z1c7hkf.html
date: '2024-07-11 20:16:25+08:00'
lastmod: '2024-08-08 13:19:17+08:00'
toc: true
tags:
  - 内存
categories:
  - 内存管理
keywords: 内存
isCJKLanguage: true
---

# 内存管理

# 内存分区

分为代码区、常量区、全局/静态区、堆区、栈区

参考进程地址空间

# 内存分配位置

参考：进程地址空间

|代码||分配位置|
| -------------------------| -------------------------------| ------------|
|char str[]|字符数组|栈或全局区|
|const char str[]|字符数组<sup>（即使字符串相同，其内存地址也不同）</sup>|栈或全局区|
|char *str="hello"|字符常量<sup>（相同字符常量在文字常量区仅保存一份，即地址相同）</sup><br />|文字常量区|
|const char *str="hello"||文字常量区|

```C++
int a = 0;  # 全局初始化区
char *p1;   # 全局未初始化区
main()
{
  int b;  # 栈
  char s[] = "abc"; # 栈(char[])
  char *p2;
  char *p3 = "123456"; # "123456"在常量区，p3在栈上。
  static int c =0； # 全局（静态）初始化区
  p1 = (char *)malloc(10);
  p2 = (char *)malloc(20); 
  # 分配得来的10和20字节的区域在堆区。
  strcpy(p1, "123456"); # 123456放在常量区，编译器可能会将它与p3所指向的"123456"优化成一个地方。
}
```

# 栈内存管理（stack）

通过{ } 限定作用域

## 堆和栈区别

||栈|堆|
| ----------| ----------------------| ----------------------------------------|
|场景|小块、频繁的内存分配|延长变量的生命周期；栈中没有足够的空间|
|生长方向|向低地址增长|向高地址增长|
|容量|比较小，如8M<sup>(Linux 上通过 ulimit -s 查看)</sup>|和虚拟内存相关|
|分配效率|软硬件<sup>（push和pop指令、rbp和rsp寄存器，）</sup>​结合优<sup>（push和pop指令、rbp和rsp寄存器）</sup>​化<sup>（push和pop指令、rbp和rsp寄存器，）</sup>，效率高||
|内存碎片|连续分配|不连续的，有内存碎片|
|分配回收|自动|手动|

例子：

* string中在栈中分配空间给小字符串（\<\=15个字符的字符串）

# 堆内存管理（new/malloc）

||new|malloc<sup>(#include <stdlib.h>)</sup>|
| ------------------| ---------------------------------------------------------------------------------------| -------------------------|
|内存大小计算|new A;|(A *)malloc(sizeof(A));|
|返回类型转换|对象类型的指针，无需进行类型转换|void*|
|函数重载|支持|不支持|
|重新分配内存|不支持|realloc(pa, N*sizeof(A))<sup>(将指针pa所指向的已分配内存区的大小拓展为N*sizeof(A))</sup>|
|分配内存并初始化|new + 调用构造函数|(A *)calloc(N, sizeof(A));<sup>(申请N*sizeof(A)大小的空间;malloc分配的内存没有初始化)</sup><br />空间初始化为0|
|内存分配失败|默认抛出异常  catch (const std::bad\_alloc& e)<br />或使用new(std::nothrow) 后判断NULL|返回NULL|
|数组分配|new[]|malloc(N*sizeof(A))<sup>(手动计算数组大小)</sup>|
|可重入||不可重入<sup>（有全局内存分配表）</sup>|
|释放内存|delete|free|

## 在已分配内存上构建对象（placement new）

特点：

* 作用：允许将object构建于allocated memory中
* 预分配内存，构建对象速度快，避免内存碎片
* 不需要delete，需要显式调用析构函数
* 可以在**栈 或 堆**上构建对象

```C++
char* buff = new char[ sizeof(Foo) * N ];
memset( buff, 0, sizeof(Foo)*N );
Foo* pfoo = new (buff)Foo;
pfoo->print();
pfoo->~Foo();
delete [] buff;

# 对象数组（构造参数带参数）
char* pong = new char[ sizeof(CPong) * 10 ];
CPong* pp = (CPong*)pong;
for ( int i=0; i<10; ++i ){
    new (pp+i)CPong(i);
}
for ( int j=0; j<10; ++j )
{
    pp[j].~CPong();
}
delete [] pong;
```

## 内存复制

```c++
# 对有析构函数等的非POD类型使用 = 拷贝构造函数
# 对保存非POD类型的数组等容器使用 std::copy,参考STL算法
int source[] = {1,2,3,4,5};
int dest[5];
std::copy(std::begin(source),std::end(source),std::begin(dest));

# 源区域和目标区域 无 重叠
void *memcpy(void *source, const void *dest, size_t n)
// n为复制的字节数

# 源区域和目标区域 可能 重叠
void *memmove(void *dest, const void *source, size_t n)
// memmove() 能够保证源串在被覆盖之前将重叠区域的字节拷贝到目标区域中，复制后源区域的内容会被更改
```

## 内存比较

* 比strncmp效率高（不检查是否结束）

```c++
int memcmp (const void *a1, const void *a2, size_t size)
// 相等返回0
```

## 内存清零

* bzero将前n个字节置0

```c++
void bzero(void *s, int n);
bzero(&serveraddr,sizeof(serveraddr));
memset(&dp,0,sizeof(dp));
```

## 内存填充

```c++
# 将dest的前n个字节填充为 val的最低一个字节
void *memset(void *dest, int val, size_t n)
memset(&dp,0,sizeof(dp)); 
```

## malloc分配原理

1. 使用内存匹配算法查看malloc的`内存池`​（多个空闲链表）中是否有合适的空闲区块
2. 默认$>=128KB$使用 **mmap**，$<128KB$使用 **sbrk**
3. ​`128KB`​可通过`mallopt()`​调整阈值`M_MMAP_THRESHOLD`​
4. 每个空闲区块**首部**有内存控制块mem_control_block，记录元信息（指向下一个分配块的指针、当前分配块的长度、或者当前区块是否已经被分配出去）

### sbrk拓展堆

```C++
#include <unistd.h>
int brk(void *addr);
void *sbrk(intptr_t increment);
```

* **sbrk**拓展堆，返回新的堆顶指针brk；
* 拓展的空闲空间交给`malloc`​管理
* sbrk分配的内存需要等到**高地址内存释放**后才能释放，会产生内存碎片
* 最高地址空间的空闲内存超过128KB（M\_TRIM\_THRESHOLD）时，执行内存紧缩操作`trim`​

### mmap匿名映射

```C++
#include <sys/mman.h>
void *mmap(void *addr, size_t length, int prot, int flags, int fd, off_t offset); // 映射磁盘文件
int munmap(void *addr, size_t length); // 匿名映射
```

* **mmap**在堆和栈之间（文件映射区域）向下拓展，而堆向上拓展
* mmap分配的内存会被**初始化**为0
* mmap分配的内存可以**单独释放**，减少内存碎片

# 虚拟内存

作用：

1. 内存保护：保护进程的空间不被其他进程破坏。
2. 简化内存管理：所有进程拥有一致的地址空间，简化了链接、加载、内存共享等过程，解决了多进程之间地址冲突的问题。
3. 进程的运行内存可以超过物理内存大小（swap）

特点：

* 虚存容量 \= min (2\^计算机位数， 内存＋外存);
* 32位计算机的内存空间为4G（字长）

## MMU内存管理单元（虚拟地址映射，访问控制）

特点：

* 作用：将虚拟地址映射为物理地址
* 检查CPU处于用户态还是内核态，以及访问内存的目的（读数据、写数据、读指令），根据权限（页表中设置页面权限为可读、可写、可执行），来决定允许访问还是产生异常

工作方式：

1. 操作系统在初始化或分配、释放内存时会执行一些指令在物理内存中**填写**页表**或段表**​，然后用指令设置MMU，告诉MMU页表或段表在物理内存中的什么位置<sup>（页表基地址寄存器）</sup>。
2. 设置好之后，CPU每次执行访问内存的指令都会自动触发MMU做查表和地址转换操作，地址转换操作由硬件自动完成，不需要用指令控制MMU去做。

## 页表（+快表TLB）

页表：分级，一般为4级页表；存储虚拟地址和物理地址的映射关系

快表：页表的高速缓存；提高地址转换速度

* 什么时候flush快表：建立映射关系flush对应的TLB表项；进程切换可能<sup>（当ASID分配完后，flush所有TLB，重新分配ASID）</sup>flush整个TLB表
* 不同进程对应相同的虚拟地址，在 TLB 是如何区分的？

  为每个进程分配一个ASID，可以区分不同的进程的TLB表项，就可以避免flush TLB

# 页面置换算法

||原理|优点|缺点|
| -------------------| --------------------------------------------------------------------------------------------| -----------------------------------| ----------------------------------------|
|最近未使用NRU|被访问设置R，被修改设置M<br />分成四类：0<sup>(没有被访问，没有被修改)</sup>、M<sup>(没有被访问，已被修改)</sup>、R<sup>(已被访问，没有被修改)</sup>、RM<sup>(已被访问，已被修改)</sup><br />随机地从类编号最小的非空类中挑选一个页面淘汰<br />|||
|先进先出FIFO||简单<br />适用于仅用一次的数据<sup>（数据库全表扫描使用环形的缓冲区）</sup>、有时效性的数据<br />|可能将重要的页换出|
|第二次机会SC|将页面换出内存前检查其使用位<sup>（使用时置1）</sup>，如果为1，将其置0后检查下一个页面|避免FIFO将重要的页换出内存||
|最近最少未使用LRU|局部性原理<sup>（果一个页面很久没有被访问，那么将来被访问的可能性也比较小。）</sup><br />|效果好|开销大；缓存污染<sup>（由于偶发性或周期性的冷数据批量查询，热点数据被换出去，导致缓存命中率下降）</sup>|
|LRU-K|最近使用过 K 次，使用历史队列<sup>（页面访问次数没有到达K次的页面，使用其他缓存策略（LRU/FIFO））</sup>和缓存队列|降低缓存污染||
|时钟CLOCK|第二次机会SC + 环形链表|避免了移动链表节点的开销||
|最不常用NFU/LFU|记录页面访问次数，优先淘汰最近访问频率最少的数据。|避免缓存污染<br />适用于**访问模式固定**的数据|访问模式变化时需要较长时间调整；开销大|
|最优OPT|将下一次使用时间离现在最长的页换出|作为衡量标准|不可实现|
|随机Random||简单<br />适用于对缓存key方向概率相等||

## LRU

原理：使用 哈希表 来记录key在list双端链表中的位置（即迭代器）

```C++
class LRUCache {
public:
    LRUCache(int capacity) {
        size = capacity;
    }
  
    int get(int key) {
        if(mp.find(key)==mp.end()) return -1;
        auto it = mp.find(key);
        cache.splice(cache.begin(),cache,it->second);
// list1.splice(position, list2, iter): 将list2中某个位置的迭代器iter指向的元素剪贴到list1中的position位置；
        return it->second->second;
    }
  
    void put(int key, int value) {
        auto it = mp.find(key);
        if(it!=mp.end()){
            it->second->second = value;
            cache.splice(cache.begin(),cache,it->second);
            return ;  
        }else{
            cache.push_front(make_pair(key,value));
            mp[key] = cache.begin();
            if(mp.size()>size){
                mp.erase(cache.back().first);
                cache.pop_back();
            }
        }
    }
private:
    list<pair<int,int>> cache;
    unordered_map<int,list<pair<int,int>>::iterator> mp;
    int size;
};

/**
 * Your LRUCache object will be instantiated and called as such:
 * LRUCache* obj = new LRUCache(capacity);
 * int param_1 = obj->get(key);
 * obj->put(key,value);
 */
```

## LFU

```C++
struct Node { // 缓存的节点信息
    int key, val, freq;
    Node(int _key,int _val,int _freq): key(_key), val(_val), freq(_freq){}
};
class LFUCache {
    int minfreq, capacity;
    unordered_map<int, list<Node>::iterator> key_table;
    unordered_map<int, list<Node>> freq_table;
public:
    LFUCache(int _capacity) {
        minfreq = 0;
        capacity = _capacity;
        key_table.clear();
        freq_table.clear();
    }
  
    int get(int key) {
        if (capacity == 0) return -1;
        auto it = key_table.find(key);
        if (it == key_table.end()) return -1;
        list<Node>::iterator node = it -> second;
        int val = node -> val, freq = node -> freq;
        freq_table[freq].erase(node);
        // 如果当前链表为空，我们需要在哈希表中删除，且更新minFreq
        if (freq_table[freq].size() == 0) {
            freq_table.erase(freq);
            if (minfreq == freq) minfreq += 1;
        }
        // 插入到 freq + 1 中
        freq_table[freq + 1].push_front(Node(key, val, freq + 1));
        key_table[key] = freq_table[freq + 1].begin();
        return val;
    }
  
    void put(int key, int value) {
        if (capacity == 0) return;
        auto it = key_table.find(key);
        if (it == key_table.end()) {
            // 缓存已满，需要进行删除操作
            if (key_table.size() == capacity) {
                // 通过 minFreq 拿到 freq_table[minFreq] 链表的末尾节点
                auto it2 = freq_table[minfreq].back();
                key_table.erase(it2.key);
                freq_table[minfreq].pop_back();
                if (freq_table[minfreq].size() == 0) {
                    freq_table.erase(minfreq);
                }
            } 
            freq_table[1].push_front(Node(key, value, 1));
            key_table[key] = freq_table[1].begin();
            minfreq = 1;
        } else {
            // 与 get 操作基本一致，除了需要更新缓存的值
            list<Node>::iterator node = it -> second;
            int freq = node -> freq;
            freq_table[freq].erase(node);
            if (freq_table[freq].size() == 0) {
                freq_table.erase(freq);
                if (minfreq == freq) minfreq += 1;
            }
            freq_table[freq + 1].push_front(Node(key, value, freq + 1));
            key_table[key] = freq_table[freq + 1].begin();
        }
    }
};
```

# 内存泄漏

1. 使用valgrind、dmalloc、efence、cppcheck等工具
2. 使用内存火焰图，截取两个时间点的内存使用情况，两者之差是可能出现内存泄漏的地方
3. VS提供`C Runtime`​调试工具
4. 编译器插件，即编译时添加`-fsanitize=address`​
5. 使用智能指针，注意避免shared_ptr循环引用
6. 使用RAII技术 或 保证异常安全
7. 重载malloc，不推荐重载operator new

    * operator new和operator delete有很多版本

      [operator new, operator new[] - cppreference.com](https://en.cppreference.com/w/cpp/memory/new/operator_new)
    * 线程安全

    [GitHub - adah1972/nvwa: My small collection of C++ utilities](https://github.com/adah1972/nvwa)

    [A Cross-Platform Memory Leak Detector (dcweb.cn)](http://wyw.dcweb.cn/leakage.htm)

# 内存分配优化

1. 使用`tcmalloc`​等内存分配器，会针对多线程场景优化
2. 使用内存火焰图查看频繁进行内存分配和释放的位置
3. 一次性malloc大块内存<sup>（减少cookie数量，减少系统调用次数）</sup>，使用freelist代替分配和回收操作，从大块内存或freelist中分配
4. 使用内存池或对象池

# 内存分配技术（存储模式）

|段式|页式|段页式|
| ----------------| ------------| ----------|
|按逻辑意义分段|页大小固定|先分段，再分页<sup>（先把程序按照逻辑意义分成段，然后每个段再分成固定大小的页）</sup>|
|没有内碎片<sup>（段大小可变，改变段大小来消除内碎片）</sup>|有<sup>（只在每个进程的最后一个页框中存在内部碎片，一个页可能填充不满）</sup>​内碎片<sup>（只在每个进程的最后一个页中存在内部碎片，一个页可能填充不满）</sup>||
|有外碎片<sup>（段换入换出时，比如4k的段换5k的段，会产生1k的外碎片）</sup>|没有外碎片<sup>（页的大小固定）</sup>||
|有利于动态链接|||

# 内存匹配算法（动态分区匹配）

## 基于顺序搜索

|算法|思想|分区排列方式|优点|缺点|
| ------------------------------| ----------------------------------| --------------------------| ------------------------------------------------------------------------------| ----------------------------------------------------|
|首次适应算法<br />First Fit|从头到尾寻找合适的分区|地址递增次序|性能好，回收分区后不需要对分区重新排序；<br />高地址保留更大的分区|低地址产生外碎片；<br />每次查找都是从低址部分开始的<br />|
|最佳适应算法<br />Best Fit|优先使用更小的分区|容量递增次序|保留更大的分区，满足大进程的需求|产生外碎片<sup>（采用内存紧凑来减少外部碎片，但移动代码和数据需要耗时）</sup>；回收分区后需要排序|
|最坏适应算法<br />最大适应算法<br />|优先使用更大的分区|容量递减次序|减少外碎片|大分区用完后，不利于大进程；回收分区后需要排序|
|临近适应算法<br />Next Fit|每次从上次查找结束的位置开始查找|地址递增次序<br />(循环链表)|性能好，回收分区后不需要对分区重新排序；<br />不需要每次从低地址的小分区开始搜索|高地址的大分区会被用完|

## 基于索引搜索

|算法|思想|优点|缺点|
| -------------------------------------| -------------------------------------------------------------------------------------------------------------| ---------------------------------------------------------------------------------------------| -----------------------------------------------------------------------------|
|快速适应算法<br />分离适配<br />Quick Fit<br />|为相同容量的分区单独设置空闲分区链表，建立索引表管理所有分区链表|不产生碎片；<br />查找效率高|回收算法时合并分区算法开销大|
|伙伴系统<br />Buddy System|分区大小必须为2的幂；<br />当需要的内存空间大于当前分区的一半的时候就将整个分区分配给进程，否则将分区对半分开<br />|不产生外碎片；<br />适合大内存分配；<br />快速搜索、快速合并;<br />申请内存大于4M，OS尽量分配连续页面<br />|满足伙伴关系的块才能合并;<br />申请内存不是2的幂会产生内碎片;<br />拆分和合并开销大|
|哈希算法|建立以空闲分区大小为关键词的哈希表|||

# 内存池

内存池管理内存块，内存块大小可能与对象大小不一致，产生内碎片

## 对象池

对象池管理某个类的对象的内存管理

# RAII

资源获取即初始化

思想：**利用栈上局部变量的自动析构来保证资源一定会被释放**。将资源和对象的声明周期绑定。

例子：lock_guard、智能指针

1. 设计一个类封装资源
2. 在构造函数中执行资源的初始化
3. 在析构函数中执行销毁操作
