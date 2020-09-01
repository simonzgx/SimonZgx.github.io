---
layout:     post
title:      "一个dynamic_cast导致的段错误"
subtitle:   "\"一个dynamic_cast导致的段错误\""
date:       2020-09-01 20:00:00
author:     "Simon"
catalog: true
header-img: "img/se-1.jpg"
tags:
   - Golang
---

> “Better code, better life. ”

## 一个dynamic_cast导致的段错误

今天写代码又出bug了

一个意料之外的段错误

下面是core dump的堆栈信息

```
Program received signal SIGSEGV, Segmentation fault.
0x00007ffff74b0796 in __cxxabiv1::__dynamic_cast (src_ptr=0x7d4050, 
    src_type=0x577a80 <typeinfo for trans_sdk::CMultiMarketDataIf>, 
    dst_type=0x577a30 <typeinfo for trans_sdk::ImplQuoteCallback>, src2dst=0)
    at ../../../../libstdc++-v3/libsupc++/dyncast.cc:68
```

很容易定位到代码在这里：

```c++
for (auto it:this->cb_) {
    auto ptr = dynamic_cast<ImplQuoteCallback *>(it.get());
    if (ptr != nullptr) {
        ptr->Update(newData);
    }
}
```

其中`cb_`的结构为：

```c++
std::vector<std::shared_ptr<CMultiMarketDataIf>> cb_;
```

在下面的函数里进行注册：

```c++
void trans_sdk::QtHandler::registerCallback(trans_sdk::CMultiMarketDataIf *cb) {
    std::shared_ptr<CMultiMarketDataIf> newCallBack(cb);
    this->cb_.emplace_back(cb);
}
```

先说一下`dynamic_cast`报段错误的几种情况：

1. 被转换的指针指向的内存已被free
2. 被转换的指针不具有多态性
3. 被转换的指针具有多态性，但有外部库所编译，并且没打开RTTI选项
4. 被转换的指针指向的内存不能被访问



错误很容易定位，审视了一下代码就能发现有一个低级错误在这里：

```c++
this->cb_.emplace_back(cb);
//应该是
this->cb_.emplace_back(newCallBack);
```

由于往一个持有只能指针的`vector`中`emplace_back`一个`raw_pointer`，会发生原为构造，即生成一个新的`shared_ptr`持有该`raw_pointer`。但在代码的上一行，已经在栈上创建了一个临时的智能指针`newCallBack`，当函数退出的时候，`newCallBack`会析构，顺便析构了其持有的`raw_pointer`所指向的内存，后面再进行`dynamic_cast`时，该指针指向的地址已经被`free`掉了，所以也就导致了段错误。