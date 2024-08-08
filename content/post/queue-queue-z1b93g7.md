---
title: queue队列
slug: queue-queue-z1b93g7
url: /post/queue-queue-z1b93g7.html
date: '2024-08-08 00:00:00+08:00'
lastmod: '2024-08-08 00:00:00+08:00'
toc: true
isCJKLanguage: true
tags:
  - 队列
categories:
  - 数据结构
keywords: 队列
---

# queue队列

# queue队列

## 底层实现

支持empty，size，front，back，push\_back，pop\_front 的容器

底层容器：deque（默认）、list

## 成员函数

|函数|功能||
| ---------| ----------| --|
|front()|头部元素||
|back()|尾部元素||
|push()|尾部插入||
|pop()|删除头部||

# priority_queue优先级队列

```C++
#include <queue>
# 最小堆，队头为最小值
priority_queue <int,vector<int>,greater<int> > pq;

pq.emplace(elem);
pq.push(elem);

pq.pop();
```

## 底层实现

支持empty, size, front, push_back, pop_back 的容器

底层容器：vector（默认，结合堆算法）、deque

# 环形队列

* rear指定队尾元素的后一位置
* 判断是否为空：front == rear
* 判断是否为满：front == (rear+1)%capacity
* 元素个数：

  用数组C[1..m]表示的环形队列，m为数组的长度。假设f为队头元素在数组中的位置，r为队尾元素的后一位置（按顺时针方向）。若队列非空，则计算**队列中元素个数**的公式应为？$( m+r-f ) % m$

```c++
class MyCircularQueue {
private:
    int front;
    int rear;
    int capacity;
    vector<int> elements;

public:
    MyCircularQueue(int k) {
        this->capacity = k + 1;
        this->elements = vector<int>(capacity);
        rear = front = 0;
    }

    bool enQueue(int value) { // 增加一个元素，在堆尾rear插入
        if (isFull()) {
            return false;
        }
        elements[rear] = value;
        rear = (rear + 1) % capacity;
        return true;
    }

    bool deQueue() { // 删除一个元素，在堆首front插入
        if (isEmpty()) {
            return false;
        }
        front = (front + 1) % capacity;
//循环队列 index+1如何处理
        return true;
    }

    int Front() {
        if (isEmpty()) {
            return -1;
        }
        return elements[front];
    }

    int Rear() {
        if (isEmpty()) {
            return -1;
        }
        return elements[(rear - 1 + capacity) % capacity];
//循环队列 index-1如何处理
    }

    bool isEmpty() {
        return rear == front;
    }

    bool isFull() {
        return ((rear + 1) % capacity) == front;
    }
};
```

# deque双端队列

* 特点：支持任意位置的元素插入和删除；支持随机访问
* 实现：分段连续空间

```C++
#include <deque>
deque<string> deq;
```

## 成员函数

|函数|功能||
| --------------| ----------| --|
|push_back()|尾部插入||
|push_front()|头部插入||
|insert()|插入||
|pop_back()|尾部删除||
|pop_front()|头部删除||
|deq[1]|随机访问||
