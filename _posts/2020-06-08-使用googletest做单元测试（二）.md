---
layout:     post
title:      "使用GoogleTest做单元测试(二)"
subtitle:   "初识GTest"
date:       2020-06-07 13:00:00
author:     "Simon"
catalog: true
tags:
    - C++11

---

> “Better code, better life. ”

# 使用`GoogleTest`做单元测试（二）

**【本文部分翻译自`GTest`[官方文档](https://github.com/google/googletest/blob/master/googletest/docs/primer.md)】**

测试并不只是测试工程师的责任，对于开发工程师，为了保证发布给测试环节的代码具有足够好的质量（ Quality ），为所编写的功能代码编写适量的单元测试是十分必要的。

**单元测试**（ Unit Test ，模块测试）是开发者编写的一小段代码，用于检验被测代码的一个很小的、很明确的功能是否正确，通过编写单元测试可以在编码阶段发现程序编码错误，甚至是程序设计错误。

单元测试不但可以增加开发者对于所完成代码的自信，同时，好的单元测试用例往往可以在回归测试的过程中，很好地保证之前所发生的修改没有破坏已有的程序逻辑。因此，单元测试不但不会成为开发者的负担，反而可以在保证开发质量的情况下，加速迭代开发的过程。

对于单元测试框架，目前最为大家所熟知的是 JUnit 及其针对各语言的衍生产品， C++ 语言所对应的 JUnit 系单元测试框架就是 `CppUnit` 。但是由于 `CppUnit` 的设计严格继承自` JUnit` ，而没有充分考虑 C++ 与 Java 固有的差异（主要是由于 C++ 没有反射机制，而这是 JUnit 设计的基础），在 C++ 中使用 `CppUnit` 进行单元测试显得十分繁琐，这一定程度上制约了 `CppUnit` 的普及。笔者在这里要跟大家介绍的是一套由 google 发布的开源单元测试框架（ Testing Framework ）： `googletest` 。

### 编译与安装

* Windows下

```bash
#下载源码
git clone https://github.com/google/googletest.git
cd googletest
#使用cmake生成vs项目文件
cmake .
```

执行完`cmake`以后，当前目录下会生成.sln和.cvxproj文件，用Visual Studio打开，然后生成指定版本的链接库

* *NIX下

```bash
#下载源码
git clone https://github.com/google/googletest.git
cd googletest
#使用cmake生成vs项目文件
cmake .
make -j
```

这里我使用的源码的commit ID为：**[4fe0180](https://github.com/google/googletest/commit/4fe018038f87675c083d0cfb6a6b57c274fb1753)**，`cmake` 3.5.1，执行完上述命令后make的时候报了很多语法错误，应该是编译时使用的C++版本问题，在`CMakeLists.txt`加入：

```cmake
set(CMAKE_CXX_STANDARD 14)
```

后一切正常。然后安装

```bash
sudo make install
```

### 使用cmake进行构建

`make install`后静态链接库声称在`/usr/local/lib`目录下，我们可以使用绝对目录来连接，当然更方便的是使用`cmake`来构建`GTest`项目，只需要在你的`CMakeLists.txt`中加入如下内容

```cmake
if (UNIX)
    find_package(Threads REQUIRED)
    find_package(GTest REQUIRED)
    if (GTest_FOUND)
        include_directories(${GTEST_INCLUDE_DIR})
    endif ()
else ()

endif ()

add_executable(${PROJECT_NAME} ${SRC_DIR})

target_link_libraries(${PROJECT_NAME} ${GTEST_LIBRARY})
target_link_libraries(${PROJECT_NAME} ${CMAKE_THREAD_LIBS_INIT})
```

下面我们来看一个使用`GTest`的例子。

### Demo1

目录结构

```bash
├── build.sh
├── CMakeLists.txt
├── gtest
├── include
│   └── Configure.h
├── main.cpp
└── src
    ├── Configure.cpp
    └── ConfigureTest.cpp
```

**Configure.h**

```c++

#ifndef GTEST_CONFIGURE_H

#define GTEST_CONFIGURE_H

#include <string>

#include <vector>

class Configure
{
private:
    std::vector<std::string> vItems;

public:
    int addItem(std::string str);

    std::string getItem(int index);

    int getSize();
};


#endif //GTEST_CONFIGURE_H
```

**Configure.cpp**

```c++
#include <algorithm>

#include "Configure.h"

/**
* @brief Add an item to configuration store. Duplicate item will be ignored
* @param str item to be stored
* @return the index of added configuration item
*/
int Configure::addItem(std::string str) {
    std::vector<std::string>::const_iterator vi = std::find(vItems.begin(), vItems.end(), str);
    if (vi != vItems.end())
        return vi - vItems.begin();

    vItems.push_back(str);
    return vItems.size() - 1;
}

/**
* @brief Return the configure item at specified index.
* If the index is out of range, "" will be returned
* @param index the index of item
* @return the item at specified index
*/
std::string Configure::getItem(int index) {
    if (index >= vItems.size())
        return "";
    else
        return vItems.at(index);
}

/// Retrieve the information about how many configuration items we have had
int Configure::getSize() {
    return vItems.size();
}
```

**ConfigureTest.cpp**

```c++
#include <gtest/gtest.h>

#include "Configure.h"

TEST(ConfigureTest, addItem)
{
    // do some initialization
    auto* pc = new Configure();

    // validate the pointer is not null
    ASSERT_TRUE(pc != nullptr);

    // call the method we want to test
    pc->addItem("A");
    pc->addItem("B");
    pc->addItem("A");

    // validate the result after operation
    EXPECT_EQ(pc->getSize(), 2);
    EXPECT_STREQ(pc->getItem(0).c_str(), "A");
    EXPECT_STREQ(pc->getItem(1).c_str(), "B");
    EXPECT_STREQ(pc->getItem(10).c_str(), "");

    delete pc;
}
```

**main.cpp**

```c++
#include <gtest/gtest.h>

int main(int argc, char** argv) {
    testing::InitGoogleTest(&argc, argv);

    // Runs all tests using Google Test.
    return RUN_ALL_TESTS();
}
```

编译运行：

```bash
cmake .
make -j
./gtest
```

**Output:**

```bash
[==========] Running 1 test from 1 test suite.
[----------] Global test environment set-up.
[----------] 1 test from ConfigureTest
[ RUN      ] ConfigureTest.addItem
[       OK ] ConfigureTest.addItem (0 ms)
[----------] 1 test from ConfigureTest (0 ms total)

[----------] Global test environment tear-down
[==========] 1 test from 1 test suite ran. (0 ms total)
[  PASSED  ] 1 test.
```

### 使用TEST宏创建测试用例

在**Demo1**的**ConfigureTest.cpp**文件中，使用了`TEST()`这个宏来创建一个Test Case，关于`TEST`宏，官方文档上主要有以下几点说明：

* 使用`TEST`来定义或者明明一个测试函数，该函数没有返回值。
* `TEST`函数和任何其他C++函数相比，只多了一点：你可以使用`GTest`提供的断言宏来对测试结果进行判断。
* 测试结果由断言确定;如果测试中的任何断言失败(致命或非致命)，或者测试崩溃，则整个测试失败。

这里我们再举一个简单的例子进行说明，有一个函数其声明为：

```c++
int Factorial(int n);  // Returns the factorial of n
```

针对这个函数的一个测试用例应该是这样的：

```c++
// Tests factorial of 0.
TEST(FactorialTest, HandlesZeroInput) {
  EXPECT_EQ(Factorial(0), 1);
}

// Tests factorial of positive numbers.
TEST(FactorialTest, HandlesPositiveInput) {
  EXPECT_EQ(Factorial(1), 1);
  EXPECT_EQ(Factorial(2), 2);
  EXPECT_EQ(Factorial(3), 6);
  EXPECT_EQ(Factorial(8), 40320);
}
```



### GTest中的断言宏

上面我们讲了，断言宏是使用在`TEST`测试用例中，用来判断测试执行结果的方法。

`GTest`的断言宏有两种：1. 形如`ASSERT_*` 2. 形如`EXPECT_*`。第一种类似`assert.h`头文件中的`assert`方法，如果表达之为false，则直接调用`abort`使程序退出；而第二种即使表达式为false，也不会退出程序，只会相应日志，继续执行其他测试用例。**官方推荐使用第二种。**

至于**\***部分，主要包括一下几种

| 类型                                  | 含义                                  |
| ------------------------------------- | ------------------------------------- |
| *_TRUE(condition)                     | condition为真                         |
| *_FALSE(condition)                    | condition为假                         |
| *_EQ(expected, actual)                | *expected*==*actual*                  |
| *_NE(val1, val2)                      | *val1*!=*val2*                        |
| *_LT(val1, val2)                      | *val1*<*val2*                         |
| *_LE(val1, val2)                      | *val1*<=*val2*                        |
| *_GT(val1, val2)                      | *val1*>*val2*                         |
| *_GE(val1, val2)                      | *val1*>=*val2*                        |
| *_STREQ(expected_str, actual_str)     | 两个 C 字符串有相同的内容             |
| *_STRNE(str1, str2)                   | 两个 C 字符串有不同的内容             |
| *_STRCASEEQ(expected_str, actual_str) | 两个 C 字符串有相同的内容，忽略大小写 |
| *_STRCASENE(str1, str2)               | 两个 C 字符串有不同的内容，忽略大小写 |

经排列组合以后，`GTest`一共有24种断言宏。

### 在不同的测试用例中共享数据

如果你需要在不同的测试用例中使用类似的数据，`GTest`提供了一种叫做`test fixture`的特性来满足你这种需求。

创建一个`fixture`：

* 创建一个继承自`::testing::Test`的类，其所有成员均为**protected**类型，因为他的成员函数/变量会在子类中被访问。
* 把你想要在多个测试用例中共享的数据在构造函数或者`SetUp()`方法中初始化。
* 如你在`SetUp()`中使用了动态内存，那么必须在声明一个`TearDown()`方法里回收这些内存。
* 使用`TEST_F(TestFixtureName, TestName)`而不是`TEST`。

**【注意】**

* `TEST_F()`的第一个参数应该是你声明的`fixture`类的类名。
* 由于C++中宏定义不允许使用单一的宏来处理所有的测试类型，所以你必须在`TEST_F()`之前定义一个`fixture`类，不然会报编译错误**virtual outside class declaration**.
* 同一个测试用例里的不同的测试具有独立的`fxiture`对象，`GTest`会在下一次测试开始前删除之前的`fixture`对象，并创建一个新的。所以，当前测试对`fxiture`的修改，不会影响下一次测试。

下面是一个`FIFO Queue`的例子：

```c++
template <typename E>  // E is the element type.
class Queue {
 public:
  Queue();
  void Enqueue(const E& element);
  E* Dequeue();  // Returns NULL if the queue is empty.
  size_t size() const;
  ...
};

//First, define a fixture class. By convention, you should give it the name FooTest where Foo is the class being tested.

class QueueTest : public ::testing::Test {
 protected:
  void SetUp() override {
     q1_.Enqueue(1);
     q2_.Enqueue(2);
     q2_.Enqueue(3);
  }
  //In this case, TearDown() is not needed since we don't have to clean up after each test, other than what's already done by the destructor.
  // void TearDown() override {}

  Queue<int> q0_;
  Queue<int> q1_;
  Queue<int> q2_;
};
```

声明完`fixture`后，我们可以在接下来的测试中使用它

```c++
//第一个参数要是上面声明的类名
TEST_F(QueueTest, IsEmptyInitially) {
  EXPECT_EQ(q0_.size(), 0);
}

TEST_F(QueueTest, DequeueWorks) {
  int* n = q0_.Dequeue();
  EXPECT_EQ(n, nullptr);

  n = q1_.Dequeue();
  ASSERT_NE(n, nullptr);
  EXPECT_EQ(*n, 1);
  EXPECT_EQ(q1_.size(), 0);
  //需要主动delete相关对象以回收资源
  delete n;

  n = q2_.Dequeue();
  ASSERT_NE(n, nullptr);
  EXPECT_EQ(*n, 2);
  EXPECT_EQ(q2_.size(), 1);
  delete n;
}
```

在上面的调用过程中，`GTest`主要做了下面几件事：

* 构造了一个`QueueTest`的实例**t1**。
* 调用`t1.SetUp()`来初始化。
* 执行第一个测试用例`IsEmptyInitially`。
* 调用`t1.TearDown()`删除实例。
* 重复以上过程，执行第二个测试用例`DequeueWorks`。

### 测试用例的执行

在**Demo1**中，我们通过`main.cpp`中的`RUN_ALL_TESTS()`宏来执行我们定义好的测试用例，但实际上，你可以连接我们第一步编译出的`gtest_main`，而不用为每一个项目写一个`main`方法。