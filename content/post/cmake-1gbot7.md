---
title: cmake
slug: cmake-1gbot7
url: /post/cmake-1gbot7.html
date: '2024-07-11 14:08:26+08:00'
lastmod: '2024-08-08 13:29:08+08:00'
toc: true
tags:
  - cmake
categories:
  - 工具
keywords: cmake
isCJKLanguage: true
---

# cmake

编译方式参考：cmake和make编译

# 预定义变量

* CMAKE\_SOURCE\_DIR：顶级CMakeLists.txt的文件夹
* PROJECT\_SOURCE\_DIR：包含最近的project()命令的CMakeLists.txt的文件夹

  > 如果项目中只有一个project()，则CMAKE\_SOURCE\_DIR和PROJECT\_SOURCE\_DIR均指项目的根目录
  >

* CMAKE\_CURRENT\_SOURCE\_DIR：当前CMakeLists.txt所在的目录
* PROJECT_BINARY_DIR：CMake生成一系列文件的目录，包括MakeFile等
* CMAKE_INSTALL_PREFIX：安装目录，默认/usr/local

# CMakeLists.txt

## 声明cmake最低版本

```c++
cmake_minimum_required( VERSION 2.8 )
```

## 项目名称

```cmake
PROJECT(test_math C CXX)
PROJECT(recipe-01 LANGUAGES CXX)
```

## 生成compile\_commands.json

```cmake
SET(CMAKE_EXPORT_COMPILE_COMMANDS True)
```

或在命令行（build.sh）中添加

```c++
-DCMAKE_EXPORT_COMPILE_COMMANDS=on
```

## 设置输出目录（+安装目录）

1. 使用cmake命令：

* cmake -DCMAKE_INSTALL_PREFIX=<install_path>

2. 在CMakeLists.txt中：

```cmake
SET(EXECUTABLE_OUTPUT_PATH ${PROJECT_BINARY_DIR}/bin)
SET(LIBRARY_OUTPUT_PATH ${PROJECT_BINARY_DIR}/lib)

SET(CMAKE_INSTALL_PREFIX <install_path>)
// 安装目录
```

## 设置头文件搜索路径

```cmake
// 头文件在/my_lib/foo.h，源文件使用 #include "my_lib/foo.h"
INCLUDE_DIRECTORIES(
	${PROJECT_SOURCE_DIR} 
)
// 源文件使用 #include "foo.h"
INCLUDE_DIRECTORIES(
	${PROJECT_SOURCE_DIR}/mylib 
)
```

## 添加文件

1. 手动指定

    ```cmake
    SET(DIR_SRCS
    	a.cpp # 当前CMakeLists.txt所在目录
    	${CMAKE_SOURCE_DIR}/src/base/b.cpp
    )
    ```

2. 使用GLOB

    CONFIGURE\_DEPENDS：添加新文件时，自动更新变量

    GLOB\_RECURSE：包含所有子文件夹

    ```cmake
    file(GLOB DIR_SRCS CONFIGURE_DEPENDS *.cpp)  # 仅搜索当前目录，不搜索子目录
    file(GLOB DIR_SRCS CONFIGURE_DEPENDS *.cpp base/*.cpp)  # +搜索base子目录
    file(GLOB_RECURSE DIR_SRCS CONFIGURE_DEPENDS *.cpp) # 包含当前目录和所有子文件夹
    ```

3. 使用AUX\_SOURCE\_DIRECTORY

    添加新文件后，cmake不知道需要重新运行cmake

    ```cmake
    AUX_SOURCE_DIRECTORY(. DIR_SRCS)
    ```

## 生成可执行文件

```cmake
ADD_EXECUTABLE(thread_test ${DIR_SRCS})  # 可执行文件名称  源文件
ADD_EXECUTABLE(${PROJECT_NAME} ${DIR_SRCS})
```

## 生成库

```cmake
ADD_LIBRARY(muduo_net ${net_SRCS})
ADD_LIBRARY(muduo_net STATIC ${net_SRCS}) # 静态库
```

## 设置构建类型

```cmake
if(NOT CMAKE_BUILD_TYPE)
  set(CMAKE_BUILD_TYPE "Debug")
endif()
set(CMAKE_CXX_FLAGS_DEBUG "-O0")
set(CMAKE_CXX_FLAGS_RELEASE "-O2 -DNDEBUG")
```

## 设置编译参数

选择一种方法，后设定的会覆盖之前的设置

1. 针对所有类型的编译器

    ```cmake
    add_compile_options(-std=c++11 -Wall -Werror)
    ```

2. ‍

    ```cmake
    set(CMAKE_CXX_STANDARD 14)
    set(CMAKE_CXX_EXTENSIONS OFF)  # 不使用编译器的拓展
    set(CMAKE_CXX_STANDARD_REQUIRED ON)  # 强制要求编译器支持所选的 C++ 标准
    ```

3. CMAKE_CXX_FLAGS

    ```cmake
    set(CXX_FLAGS
     -g
     # -DVALGRIND
     -DCHECK_PTHREAD_RETURN_VALUE
     -D_FILE_OFFSET_BITS=64
     -Wall
     -Wextra
     -Werror
     -Wconversion
     -Wno-unused-parameter
     -Wold-style-cast
     -Woverloaded-virtual
     -Wpointer-arith
     -Wshadow
     -Wwrite-strings
     -march=native
     # -MMD
     -std=c++11
     -rdynamic
     )
    if(CMAKE_BUILD_BITS EQUAL 32)
      list(APPEND CXX_FLAGS "-m32")
    endif()
    if(CMAKE_CXX_COMPILER_ID STREQUAL "Clang")
      list(APPEND CXX_FLAGS "-Wno-null-dereference")
      list(APPEND CXX_FLAGS "-Wno-sign-conversion")
      list(APPEND CXX_FLAGS "-Wno-unused-local-typedef")
      list(APPEND CXX_FLAGS "-Wthread-safety")
      list(REMOVE_ITEM CXX_FLAGS "-rdynamic")
    endif()
    string(REPLACE ";" " " CMAKE_CXX_FLAGS "${CXX_FLAGS}")
    # 将 CXX_FLAGS 变量中的所有分号替换为空格，并将替换后的结果存储到 CMAKE_CXX_FLAGS 变量中
    ```

4. add_library 后设置库的编译参数

    ```cmake
    add_library(muduo_protobuf_codec ProtobufCodecLite.cc)
    set_target_properties(muduo_protobuf_codec PROPERTIES COMPILE_FLAGS "-Wno-error=shadow")
    ```

## 设置库或可执行文件的链接库

```cmake
target_link_libraries(muduo_base pthread rt)
target_link_libraries(asynclogging_test muduo_base)
```

## 添加测试

参考：gtest

## 引入第三方库

指向本地已经下载好的代码，参考软件下载位置

```cmake
# 包含 FetchContent 模块，用于从外部资源获取依赖项
include(FetchContent)
FetchContent_Declare(
  	googletest
	# SOURCE_DIR ./googletest
	# 使用上面一句来指向本地已经下载好的代码
  	GIT_REPOSITORY https://github.com/google/googletest.git
  	# GIT_TAG release-1.10.0
	GIT_TAG origin/main
)
FetchContent_MakeAvailable(googletest)
include(GoogleTest)
```

## 输出信息

```cmake
MESSAGE( STATUS "src = ${DIR_SRCS}.")
```

## 添加子目录

* 子目录也需要一个CMakeLists.txt
* 路径为当前CMakeLists.txt的相对路径

```cmake
add_subdirectory(muduo/base)
add_subdirectory(muduo/net)
```

## 安装库（+头文件）

* CMAKE_INSTALL_PREFIX默认为 /usr/local，修改安装目录参考设置输出目录（+安装目录）
* 仅安装用户可见的头文件
* 可以指定安装文件的权限 PERMISSIONS
* 可以指定install 适用于DEBUG/RELEASE (CONFIGURATIONS)
* 可以指定安装的文件不存在也不报错 OPTIONAL

```cmake
install(TARGETS muduo_net DESTINATION lib)
# 在CMAKE_INSTALL_PREFIX/bin 目录下安装库
install(FILES ${HEADERS} DESTINATION include/muduo/net)
# 安装头文件
```

## 变量SET

```cmake
set(base_SRCS
  AsyncLogging.cc
  Condition.cc
)
# set仅在本CMakeLists.txt可见

SET(variable value CACHE INTERNAL "")
# 设置全局变量
```
