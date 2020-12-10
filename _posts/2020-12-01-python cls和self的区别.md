---
layout:     post
title:      "C++和python函数调用的区别与联系"
subtitle:   "\"C++和python函数调用的区别与联系\""
date:       2020-12-06 12:00:00
author:     "Simon"
catalog: true
header-img: "img/Earth-2K-Wallpaper.jpg"
tags:
   - Python

---

> 时隔两周，加入字节跳动后的第一篇技术博客

## 从程序的执行说起

在聊这个话题之前，我们需要先知道，在一个程序中，每一个函数/方法是如何被调用/执行的。

现在的程序语言，无论是编译型如C++，还是解释型如python,java，最终被CPU所执行的都是一行行机器码。

现代CPU内部有很多种寄存器，这里我们只说指令寄存器（IR）和程序计数器（PC）。

对于一段程序，我们拿C++来举例，当操作系统

## 一般方法

## 类方法

目前对于类方法的调用，各个语言主要有两种方式

（这里所说的显示和隐式的主要是说对于类方法，`this`指针的声明方式）

#### C++的隐式调用

在C++中，使用操作符`.`或者`->`来访问一个类的成员变量或者成员方法。如：

```c++
class Foo{
  
public:
  static void bar(int val){}
  
  void func(int val){}

};

int main(){
  Foo::bar(1);
  Foo foo;
  foo.func(2);
  // auto foo = new Foo(3);
  // foo->func(4);
}
```

### python的显示调用

```python
class Foo{
  def func(self, val):
  	pass
  
  @classmethod
  def bar(cls, val):
  	pass
}
```

可见，相对于C++，python的每一个成员方法都显式的指明了`self` 或`cls`  。但这只是声明方式的不同，起本质依然是一样的。

