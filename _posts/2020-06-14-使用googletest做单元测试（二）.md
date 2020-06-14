---
layout:     post
title:      "使用GoogleTest做单元测试(二)"
subtitle:   "初识GTest"
date:       2020-06-14 13:00:00
author:     "Simon"
catalog: true
tags:
    - C++11

---

> “Better code, better life. ”

# 使用`GoogleTest`做单元测试（二）

**【本文接上】**

### 更多测试断言

* **Success**和**Failure**

`SUCCEED()`和`FAIL()`断言不会做任何测试，只是产生一个成功或失败的信号。

`SUCCEED()`并不会让整个`TEST`都返回成功，它只是返回当前测试分支的成功。【注意】`TEST`通过的条件是，所有的断言都通过！

`FAIL()`会产生一个**致命错误**，而类似的`ADD_FAILURE()`和`ADD_FAILURE_AT()`产生**非致命错误**，并且可以附带错误信息。这里所谓的致命和非致命类似前文讲过的`ASSERT`，会让当前`TEST`直接结束。

使用`SUCCEED()`和`FAIL()`断言可以让我们更好的控制测试案例的执行流程，比如有如下代码：

```c++
switch(expression) {
  case 1:
     ... some checks ...
  case 2:
     SUCCEED()
  default:
     FAIL() << "We shouldn't get here.";
}
```

* **异常判断宏**

对于一段代码是否会抛出异常，`GTest`提供了一下几个断言宏

| 类型                                     | 说明                            |
| ---------------------------------------- | ------------------------------- |
| ASSERT_THROW(statement, exception_type); | statement抛出对应type的异常     |
| ASSERT_ANY_THROW(statement);             | statement抛出任何类型type的异常 |
| ASSERT_NO_THROW(statement);              | statement不抛出任何类型的异常   |
| EXPECT_THROW(statement, exception_type); | statement抛出对应type的异常     |
| EXPECT_ANY_THROW(statement);             | statement抛出任何类型type的异常 |
| EXPECT_NO_THROW(statement);              | statement不抛出任何类型的异常   |

举个栗子：

```c++
ASSERT_THROW(Foo(5), bar_exception);

EXPECT_NO_THROW({
  int n = 5;
  Bar(&n);
});
```

* **更方便的检查返回bool类型的函数**

`GTest`提供了一种方法能更好的测试返回bool类型的函数，先看例子:

```c++
// Returns true if m and n have no common divisors except 1.
bool MutuallyPrime(int m, int n) { ... }

const int a = 3;
const int b = 4;
const int c = 10;
```

以我们之前的经验，对于函数`MutuallyPrime`的测试可能是这样的：

```c++
EXPECT_EQ(MutuallyPrime(a,b), true);
```

更便捷的做法是这样：

```c++
EXPECT_PRED2(MutuallyPrime, a, b); //succeed
EXPECT_PRED2(MutuallyPrime, b, c); //fail
```

`_PRED*`这个断言宏是专门为类似的功能所涉及的，一共有如下几种：

| 类型                            |
| ------------------------------- |
| ASSERT_PRED1(pred1, val1)       |
| ASSERT_PRED2(pred2, val1, val2) |
| EXPECT_PRED1(pred1, val1)       |
| EXPECT_PRED2(pred2, val1, val2) |

*****所代表的是参数的个数。

* **AssertionResult**

**AssertionResult**是`GTest`提供的一种特殊的类型，断言结果类型（可能是`SUCCED`或者`FAIL`等等）。同时`GTest`也提供了一个工厂方法来创建**AssertionResult**类型：

```c++
namespace testing {

// Returns an AssertionResult object to indicate that an assertion has
// succeeded.
AssertionResult AssertionSuccess();

// Returns an AssertionResult object to indicate that an assertion has
// failed.
AssertionResult AssertionFailure();

}
```

**AssertionResult**类型重载了`<<`操作符，所以你可以很方便的追加异常信息，我们可以比较以下两种方式：

```c++
::testing::AssertionResult IsEven(int n) {
  if ((n % 2) == 0)
     return ::testing::AssertionSuccess();
  else
     return ::testing::AssertionFailure() << n << " is odd";
}
```

和

```c++
bool IsEven(int n) {
  return (n % 2) == 0;
}
```

以上两个方法，对于`EXPECT_TRUE(IsEven(Fib(4)))`的输出结果分别为

Output1:

```c++
Value of: IsEven(Fib(4))
  Actual: false (3 is odd)
Expected: true
```

Output2:

```c++
Value of: IsEven(Fib(4))
  Actual: false
Expected: true
```

显然，第一种更具有可读性。

* **类型断言**

你可以使用以下方法：

```c++
::testing::StaticAssertTypeEq<T1, T2>();
```

来断言`T1`和`T2`的类型是否一致。如果不一致，则会产生`FATAL ERROR`，来结束该测试用例。

**【注意】**当时试图对一个模板函数进行该断言时，当且仅当该函数被实例化后，才有效果。比如:

```c++
template <typename T> class Foo {
 public:
  void Bar() { ::testing::StaticAssertTypeEq<int, T>(); }
};

#void Test1() { Foo<bool> foo; } //错误！不能断言未被实例化的模板函数
void Test2() { Foo<bool> foo; foo.Bar(); } //正确
```

### 重载`GTest`的默认断言结果输出函数

当我们的每个断言成功或者失败的时候，都会有相应信息输出再屏幕上，来帮你做debug。默认的`printer`能对C++内置类型、STL容器都可以做输出，而对于自定义类型，需要你自己重载相应的输出函数。

比如：

```c++
#include <ostream>

namespace foo {

class Bar {  // We want googletest to be able to print instances of this.
...
  // Create a free inline friend function.
  friend std::ostream& operator<<(std::ostream& os, const Bar& bar) {
    return os << bar.DebugString();  // whatever needed to print bar to os
  }
};

// If you can't declare the function in the class it's important that the
// << operator is defined in the SAME namespace that defines Bar.  C++'s look-up
// rules rely on that.
std::ostream& operator<<(std::ostream& os, const Bar& bar) {
  return os << bar.DebugString();  // whatever needed to print bar to os
}

}  // namespace foo
```

再以上实例代码中，针对`Bar`类型，重载了`<<`操作符，如果`Bar`类型本身就重载了`<<`操作符，我们只需简单的定义一个`PrintTo*`方法，就像这样：

```c++
#include <ostream>

namespace foo {

class Bar {
  ...
  friend void PrintTo(const Bar& bar, std::ostream* os) {
    *os << bar.DebugString();  // whatever needed to print bar to os
  }
};

// If you can't declare the function in the class it's important that PrintTo()
// is defined in the SAME namespace that defines Bar.  C++'s look-up rules rely
// on that.
void PrintTo(const Bar& bar, std::ostream* os) {
  *os << bar.DebugString();  // whatever needed to print bar to os
}

}  // namespace foo


TEST(T1, t){
    vector<pair<Bar, int> > bar_ints = GetBarIntVector();

	EXPECT_TRUE(IsCorrectBarIntVector(bar_ints))
    	<< "bar_ints = " << ::testing::PrintToString(bar_ints);
}
```

