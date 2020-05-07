---
layout:     post
title:      "Windows下安装配置Boost库"
subtitle:   " \"c++11\""
date:       2020-05-07 12:00:00
author:     "Simon"
catalog: true
tags:
    - Boost

---

> “Better code, better life. ”

# Windows下安装配置Boost库

#### 安装gcc

* 下载[MinGW](https://osdn.net/projects/mingw/releases/)
* 安装到**C:\Program Files (x86)\mingw-w64**
* 将**C:\Program Files (x86)\mingw-w64\i686-8.1.0-posix-dwarf-rt_v6-rev0\mingw32\bin**添加到环境变量`Path`中，否则下面编译`Boost`库时会报找不到`g++`的错误。
* 打开`cmd`或者`powershell` 输入`g++ -v`验证

#### 安装Boost

* 下载[Boost](https://dl.bintray.com/boostorg/release/1.73.0/source/)
* 解压缩到任意目录后，在解压目录下`shift+右键`打开`cmd`或者`powershell`。
* 执行`.\bootstrap.bat mingw`
* 执行结束后当前目录会生成一个**b2.exe**文件。
* 执行`.\b2.exe install`开始编译。
* 安装目录默认为**C:\Program Files\boost**，也可以在上面的命令后加`.\b2.exe install --prefix="C:\Program Files\boost-build"`改变安装目录。

#### 测试Boost

`CMakeLists.txt`内容：

```c++
cmake_minimum_required(VERSION 3.15)

project(test_boost)

# 以下三个编译选项为可选
set(Boost_USE_STATIC_LIBS ON)
set(Boost_USE_MULTITHREADED ON)
set(Boost_USE_STATIC_RUNTIME OFF)

find_package(Boost)

set(INCLUDE_DIR include)
    
if(Boost_FOUND)
    list(APPEND INCLUDE_DIR ${Boost_INCLUDE_DIRS})
    include_directories(${Boost_INCLUDE_DIRS})
    link_directories(${Boost_LIBRARY_DIRS})
else()
    message(STATUS "Find Boost_Lib error")
endif()

aux_source_directory(. SRC_DIR)

include_directories(${INCLUDE_DIR})

add_executable(test_boost ${SRC_DIR})

target_link_libraries(test_boost ${Boost_LIBRARIES})
```

源文件`main.cpp`内容为：

```c++
#include <iostream>
#include <boost/asio.hpp>

using namespace std;

int main() {
    cout<<"hello"<<endl;
}
```

Output:

```
hello
```

编译通过说明`Boost`库已经可以正常使用！

