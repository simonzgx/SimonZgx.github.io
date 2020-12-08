---
layout:     post
title:      "python cls和self的区别"
subtitle:   "\"python cls和self的区别\""
date:       2020-12-06 12:00:00
author:     "Simon"
catalog: true
header-img: "img/Earth-2K-Wallpaper.jpg"
tags:
   - Python

---

### 类方法

目前对于类方法的调用，各个语言主要有两种方式

#### C++的隐式调用

在C++中，使用操作符`.`或者`->`来访问一个类的成员变量或者成员方法。如：

```c++
class Foo{
  
public:
  static void bar(){}
  
  void func(){}

};

int main(){
  Foo::bar();
  Foo foo;
  foo.func();
  // auto foo = new Foo();
  // foo->func();
}
```

