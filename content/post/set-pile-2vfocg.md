---
title: set堆
slug: set-pile-2vfocg
url: /post/set-pile-2vfocg.html
date: '2024-07-09 10:57:10+08:00'
lastmod: '2024-08-08 13:26:44+08:00'
toc: true
tags:
  - 堆
categories:
  - 数据结构
keywords: 堆
isCJKLanguage: true
---

# set堆

* **==完全二叉树==**​（不是平衡二叉树）
* 可以使用数组实现
* 任意节点的值是其子树所有节点的最值
* 小根堆中最大的数一定是放在叶子节点上<sup>（堆本身是个完全二叉树，完全二叉树的叶子节点的位置大于[n/2]）</sup>

# 数组实现

​![image](https://raw.githubusercontent.com/arukasxy/notes/main/content/post/image-viewer/image-20240709110529-nyezep6.png)​![image](https://raw.githubusercontent.com/arukasxy/notes/main/content/post/image-viewer/image-20240709110542-2l5ixt7.png)​

* 当数组从0开始时，下标为k的结点的父结点下标为$(k-1)/2$；左儿子下标为$2$​*$i+1$*​ *，右儿子下标为*​*$2$*​$i+2$。

* 当数组从1开始时，下标为k的结点的父结点下标为$k/2$；左儿子下标为$2$​*$i$*​ *，右儿子下标为*​*$2$*​$i+1$。

# 复杂度

建堆时间复杂度：==$O(n)$==  

堆排序时间复杂度：==$O(nlogn)$==

# STL堆算法

* pop_heap 将堆的第零个元素和最后一个元素交换位置，再针对前n-1个元素调用make\_heap()函数,注意并没有移除元素

```c++
#include <algorithm>
# 将可迭代容器（如vector）变成堆（默认大根堆less）
std::make_heap( q.begin(), q.end(), less<int>() )

# 将数据插入堆中
q.push_back(val);
std::push_heap( q.begin(), q.end() );

# 弹出数据
std::pop_heap( q.begin(), e.end() )
q.pop_back()
```

# 堆实现

以下为大根堆，将<改成>即为小根堆

```c++
# 从下标1开始存储数据，如果下标从0，参考向下调整中的代码
class Heap {
private:
	vector<int> a; // 数组
	int n;  // 堆可以存储的最大数据个数
	int count; // 堆中已经存储的数据个数

public:
	Heap(int capacity) {
		a.resize( capacity + 2);
    	n = capacity;
    	count = 0;
	}

	void insert(int data) {
		++count;
    	a[count] = data;
    	int i = count;
    	while (i/2 > 0 && a[i] > a[i/2]) { // 自下往上堆化
      		swap(a[i], a[i/2]);
      		i = i/2;
		}
		if( count>n ){ //堆满了
			removeMax();
		}
	}

	void removeMax() {
		if (count == 0) return ; // 堆中没有数据
		a[1] = a[count];
		--count;
		heapify(a, count, 1);
	}

	void heapify(vector<int>& a, int n, int i) { // 自上往下堆化
		while (true) {
    		int maxPos = i;
    		if (i*2 <= n && a[i] < a[i*2]) maxPos = i*2;
    		if (i*2+1 <= n && a[maxPos] < a[i*2+1]) maxPos = i*2+1;
    		if (maxPos == i) break;
    		swap(a[i], a[maxPos]);
    		i = maxPos;
		}
	}

	vector<int> Get(){
		return {a.begin()+1,a.begin()+count+1};  //返回数据
	}
}
```

## 向下调整

前提：根结点的左右子树均为大堆或是小堆

场景：删除元素

1. 从根结点处开始，选出左右孩子中值较小的孩子。
2. 让小的孩子与其父亲进行比较。
3. 若小的孩子比父亲还小，则该孩子与其父亲的位置进行交换。并将原来小的孩子的位置当成父亲继续向下进行调整，直到调整到叶子结点为止。

```C++
# 下标从0开始，如果从1开始，参考class Heap
# 只调整下标为i的元素，堆的大小为n
void heapAdjustDown(vector<int>& a, int i,int n) { // 自上往下堆化
    while (true) {
        int maxPos = i;
        if (i*2+1 < n && a[i] < a[i*2+1]) maxPos = i*2+1;
        if (i*2+2 < n && a[maxPos] < a[i*2+2]) maxPos = i*2+2;
        if (maxPos == i) break;
        swap(a[i], a[maxPos]);
        i = maxPos;
    }
}
```

## 将树调整为堆

从倒数第一个非叶子结点开始，从后往前，按下标，依次作为根去向下调整即可（只能使用向下调整算法）

```C++
# 构建堆,下标从0开始
int n = nums.size();
for (int i = n/2-1; i >= 0; i--) { // 从最后一个非叶节点开始，不断向下调整
    heapAdjustDown(nums, i, n);
}
```

## 向上调整

场景：插入元素

1. 先将插入元素放在最后，再逐步和父节点交换

```C++
# 向上调整
void headAdjustUp(vector<int>& nums, int s) {
    int t = nums[s];
    // i 表示当前节点，手动计算父节点
    for (int i = s; i > 0; i = (i - 1) / 2) {
        if (nums[(i-1)/2] < t) {
            nums[s] = nums[(i-1)/2];
            s = (i - 1) / 2;
        } else {
            break;
        }
    }
    nums[s] = t;
}
```

## 删除元素

先将堆顶的数据与最后一个结点的位置交换，然后再删除最后一个结点，再对堆进行一次向下调整。

# set

特点：

* 内部元素不重复
* 可以有序，默认从小到大

```C++
#include<set>

# 删除元素
s.erase(k);
s.clear();

# 最值
item = *s.begin();
```

# unordered_set

特点：

* 快速判断集合是否存在某元素
* 原理：哈希表

```C++
unordered_set<string> s(vec.begin(), vec.end());
unordered_set<string> s{str1};

s.insert(element);
```

# bitset 位图

* 需要编译时确定大小，如果动态开空间，则可以用`vector<bool>'
* 参考基础位运算

```C++
#include <bitset>

bitset<100000> bs; // 初始化为0

# 使用字符串初始化前n位
bitset<16> bs3(string("10111001"));

bitset<32> bs(unsigned long val);
bitset<32> bs(bitset);

# 可直接赋值
bs[3]=1;

# 可以进行比较
bs1 == bs2
```

## 成员函数

|函数|功能||
| ----------| ---------------------| --|
|test(k)|获取第k位的值||
|set(k,v)|将第k位设置为v||
|set()|所有位置1||
|reset()|所有位置0||
|reset(k)|将第k位置0||
|count()|1的个数||
|any()|至少一个1，返回true||
|none()|全为0，返回true||
|flip()|所有位取反||
|flip(k)|将第k位取反||
