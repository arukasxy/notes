---
title: 文件
slug: document-z112xbv
url: /post/document-z112xbv.html
date: '2024-07-14 12:07:08+08:00'
lastmod: '2024-08-08 13:17:18+08:00'
toc: true
tags:
  - 文件
categories:
  - 文件
keywords: 文件
isCJKLanguage: true
---

# 文件

# inode元信息

相关inode命令参考inode

* 硬盘格式化时分为两个区域：数据区、**inode**区
* inode节点大小为**128B 或 256B**，节点总数在格式化时已经确定
* 每个文件都**必需1个inode**，可能出现inode已经用完，但磁盘没满的情况

# 文件描述符

* **多个文件描述符**可以指向同一个打开文件
* int类型
* **进程控制块PCB**中有指向进程**打开文件**的**文件描述符表**
* 内核有一个**系统级**的打开文件表

## 特殊文件描述符

|含义|FILE*<sup>(由fread、fwrite、fclose等函数使用)</sup>|int<sup>(由read、write、close等函数使用)</sup>|
| ----------| --------| ---------------|
|标准输入|stdin|STDIN_FILENO|
|标准输出|stdout|STDOUT_FILENO|
|标准错误|stderr|STDERR_FILENO|

## 复制或重定向文件描述符

```C++
#include <unistd.h>
# 返回指向oldfd的新文件描述符
int dup(int oldfd);

# 返回指向oldfd的newfd, 如果newfd已打开文件，则先关闭该文件;如果newfd==oldfd，则不会关闭
int dup2(int oldfd, int newfd);

# 将输入重定向到fd中
dup2(fd, STDIN_FILENO);

# 保存文件描述符
int saveStdout = dup(STDOUT_FILENO);
```

# 打开文件

* 文件对象与磁盘文件建立联系

原理：

1. 找到文件名对应的 inode号
2. 通过 inode号，获取 inode元信息
3. 进行权限判断，找到文件数据所在的block，读取数据

```C++
# 文件流, std::fstream 可读写, ofstream 写, ifstream读
ofstream file;  
file.open (文件路径, ios::out | ios::app | ios::binary);
if (!file.is_open()){
	...
}

# 带缓冲
FILE *  fopen (const char  *filename, const  char  *mode);
FILE *fp;
if((fp=fopen(filename,"r")) == NULL){
	...
}

# 无缓冲
int open(const char * pathname, int flags, mode_t mode);
```

|含义|stream|fopen|open|
| --------------------------| ----------------| -------| ----------------|
|读|ios::in|r|O_RDONLY|
|写|ios::out|w|O_WRONLY|
|读写||+|O_RDWR|
|追加|ios::app|a|O_APPEND|
|文本文件|默认|t||
|二进制文件|ios::binary|b||
|文件存在先清空|ios::trunc||O_TRUNC|
|文件不存在报错|ios::nocreate|||
|文件存在报错<br />+ 创建文件|ios::noreplace||O_EXCL<sup>(EEXIST 文件已存在)</sup> \| O_CREAT|
|初始位置为文件尾|ios::ate|||
|关闭子进程fork得到的fd|||O_CLOEXEC|

# 关闭文件

* 释放内存中的文件对象，保证将输出的数据写入硬盘

```C++
# 关闭文件流
fstreamFile.close();

int fclose(FILE *stream);

close(int fd)
```

# 删除文件

* remove 调用rmdir删除目录

* remove 调用unlink删除文件

```C++
#include <stdio.h>
int remove(const char *pathname)
const char *file_name = "test.txt";
if( file_name != nullptr ){
	if (remove(file_name) == 0) {
      // 成功删除文件
	} else {
      // 失败
	}
}
```

# 修改文件名称

```C++
#include <cstdio>
int ret = std::rename("old.txt", "new.txt");
// old文件名必须已经存在，new文件名必须不存在
if( ret!= 0){
	return RC::IOERR_WRITE； // 返回错误码
}
```

# 修改文件属性

```C++
# 修改已打开文件的属性
#include <unistd.h>
#include <fcntl.h>

int fcntl(int fd, int cmd);
int fcntl(int fd, int cmd, long arg);
int fcntl(int fd, int cmd, struct flock *lock);

# 设置非阻塞
int flag = fcntl(fd, F_GETFL, 0);
fcntl(fd, F_SETFL, flag|O_NONBLOCK)
```

# 读写指针

```C++
# 获取读指针（返回int）
int pos = tellg();

# 设置读指针
istream & seekg (int offset, int mode);
ioFile.seekg(0,ios::end);

# 获取写指针
int pos = tellp();

# 设置写指针
ostream & seekp (int offset, int mode);
ioFile.seekp(0,ios::end);

#include <sys/types.h>
#include <unistd.h>

# 获取读写指针
int len = ftell(fp);

# 设置读写指针
off_t lseek(int fd, off_t offset, int whence);
off_t currpos;
currpos = lseek(fd, 0, SEEK_CUR);
```

|基址|offset偏移|流stream|C|
| ----------| ------------| ----------| --------------|
|文件开始|非负数0+|ios::beg|SEEK\_SET|
|当前位置||ios::cur|SEEK\_CUR|
|文件结尾|非正数0-|ios::end|SEEK\_END|

# 刷新缓冲区（强制写回）

```C++
# 换行，然后刷新缓冲区
cout<<“hi!”<<endl;
# 刷新缓冲区，不添加任何额外字符
cout<<“hi!”<<flush; 
# 输出一个空字符，然后刷新缓冲区
cout<<“hi!”<<ends; 

# 如果参数stream为NULL， fflush()会将所有打开的文件数据更新。
int fflush(FILE *stream)
```

# 空洞文件

* 没有写过的字节都被设为0
* 原理：文件位移量可以大于文件的当前长度，对该文件的下一次写将延长该文件，并在文件中构成一个空洞
* 作用：

  * 下载文件时先占位
  * 文件下载时从不同位置多线程写入

```C++
#include <sys/types.h>
#include <sys/stat.h>
#include <fcntl.h>
#include <stdio.h>
#include <unistd.h>
 
int main()
{
    char *pathname = "./test.data";
    long long length = (long long)1024 * 1024 * 1024; //1GB
 
    //创建文件
    int fd = open(pathname, O_RDWR | O_CREAT | O_EXCL, 0777);
 
    long long ret = lseek(fd, length, SEEK_END);
 
    write(fd, "0", 1);
    close(fd);
 
    return 0;
}
```

# mmap内存映射文件

特点：

* 作用：将硬盘文件映射到进程的地址空间（虚拟地址）
