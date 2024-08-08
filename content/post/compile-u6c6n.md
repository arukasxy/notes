---
title: 编译
slug: compile-u6c6n
url: /post/compile-u6c6n.html
date: '2024-07-12 16:06:32+08:00'
lastmod: '2024-08-08 13:09:42+08:00'
toc: true
tags:
  - 编译
categories:
  - 编译
keywords: 编译
isCJKLanguage: true
---

# 编译

# 编译命令

1. 单文件编译

    ```shell
    g++ test.cpp -o test -fsanitize=address
    ```

2. cmake和make编译

    设置安装目录：设置输出目录（+安装目录）

    ```shell
    ./configure
    cmake -S . -B build
    # 或使用 cd build  +   cmake ../
    cd build
    make -sj 2 2>&1|grep rror
    # -B always-make 总是重新构建所有
    # -d | more 打印调试信息
    # -n 预演make，但不实际执行，可以将其重定向到一个文件中
    # --DDEBUG=ON debug模式
    # -j4 多线程编译
    # -s 静默安装
    make install -sj 16
    ctest 或 ./hello_test
    # 执行测试, ctest需add_test
    make unistall
    # 卸载安装的程序
    make clean
    # 删除make产生的文件
    make distclean
    # 删除由./configure产生的文件
    ```

# 编译脚本build.sh

```sh
#!/bin/sh

set -x

SOURCE_DIR=`pwd`  # 只能以./build.sh执行，不能在文件夹外执行
BUILD_DIR=${BUILD_DIR:-../build}
BUILD_TYPE=${BUILD_TYPE:-release}
INSTALL_DIR=${INSTALL_DIR:-../${BUILD_TYPE}-install-cpp11}
CXX=${CXX:-g++}

ln -sf $BUILD_DIR/$BUILD_TYPE-cpp11/compile_commands.json

mkdir -p $BUILD_DIR/$BUILD_TYPE-cpp11 \
  && cd $BUILD_DIR/$BUILD_TYPE-cpp11 \
  && cmake \
           -DCMAKE_BUILD_TYPE=$BUILD_TYPE \
           -DCMAKE_INSTALL_PREFIX=$INSTALL_DIR \
           -DCMAKE_EXPORT_COMPILE_COMMANDS=ON \
           $SOURCE_DIR \
  && make $*
```

# 优化级别

```shell
g++ -c -Q -O2 --help=optimizers > /tmp/O2-opts
# 查看O2的优化项
g++ -c -Q -O0 --help=optimizers > /tmp/O-opts
# 查看O0的优化项
diff /tmp/O2-opts /tmp/O-opts | grep enabled
# 查看O2相比O0多的优化项
```

|优化级别|优化项|影响|
| -------------------| -----------------------------| ----------------------------|
|O0|不做任何优化||
|O1|死代码消除、尾调用优化|基本不影响调试|
|O2|指令调度、自动内联函数|目标代码和源代码不一一对应|
|O3|高级标量优化、更激进的内联||

# 编译选项

|选项|作用|
| --------------------| -------------------------------------------|
|-std=c++2a|C++20|
|-D DEBUG|设置DEBUG宏|
|-g|生成调试信息|
|-g3|生成可以调试宏的信息（release也可以使用）|
|-O3|设置优化级别|
|-Wall|启用额外的警告信息|
|-ltbb|链接tbb库|
|-fsanitize=address|检测内存泄漏|

# 编译流程

1. **预处理**：宏展开、头文件包含、条件编译（#if）、删除注释

    ```shell
    g++ -E hello.cpp > hello.i
    ```

2. **编译**：语法分析、语义分析、中间代码生成、优化、目标代码生成（汇编代码  **.s**）

    ```shell
    g++ -S hello.cpp
    g++ -S hello.i
    ```

3. **汇编**：将汇编代码转为**二进制代码（.o）**

    ```shell
    g++ -c hello.cpp
    g++ -c hello.i
    g++ -c hello.s
    ```

4. **链接**：符号解析（确保符号有定义且唯一）、地址重定向（到最终的地址空间）、将**目标文件**和**库文件**链接，生成可执行文件

    ```shell
    g++ hello.cpp
    g++ hello.cpp -o hello
    g++ hello.o -o hello
    ```

# 静态库和动态库

## 静态库（.lib/.a）

优点：

* 编译时链接，依赖管理在编译期决定，调试core dump不会遇到库更新导致debug符号失效情况
* 运行速度更快，因为没有过程查找表PLT，函数调用开销更小
* 发布方便，仅需拷贝单个可执行文件

缺点：

* 应用程序与静态库（或静态库与静态库）依赖冲突<sup>（使用版本不同的库）</sup>
* 静态库需要为底层库（如g++）的变动编译并发布新的库

## 动态库（.so/.dll）

优点：

* 运行时链接
* 动态更新（hot fix bug）
* 节省磁盘和内存空间

缺点：

* 需要部署和**更新**动态库，如何确保所有机器上的动态库被更新，如何确保不会损坏现有的应用程序

# 设置编译器路径

```shell
# 需要指定到bin目录
export PATH="$PATH:/opt/au1200_rm/build_tools/bin"
# 推荐
source ~/.bashrc
```

# 动态库路径

1. ​`DT_RPATH`​：编译时指定-rpath
2. ​`LD_LIBRARY_PATH`​和`LIBRARY_PATH`​
3. ​`DT_RUNPATH`​
4. ​`/etc/ld.so.cache`​：动态加载时由ldconf生成的cache
5. ​`/lib`​和`/usr/lib`​

```shell
g++ -o main main.o -lPocoFoundation -Wl,-rpath,/opt/mker/poco/lib

# 静态链接
g++ test.cpp -o test /home/gcc10.3/gcc/lib64/libstdc++.a
```

## 可执行文件的动态链接库

```shell
ldd /bin/ls
```

## 问题解决

```shell
# 如果提示error while loading shared libraries: libisl.so.15: cannot open shared object file
export LD_LIBRARY_PATH=/xxxx/isl/lib:$LD_LIBRARY_PATH

# 如果提示libstdc++.so.6: version \`CXXABI\_1.3.11‘ not found
# 找到libstdc++.so的路径
export LIBRARY_PATH="/home/buildtools/gcc10.3/gcc/lib64/:$LIBRARY_PATH"
export LD_LIBRARY_PATH="$LIBRARY_PATH:$LD_LIBRARY_PATH"
```
