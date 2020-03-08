---
layout:     post
title:      "手撕数据结构---从priority_queue到heap"
subtitle:   " \"数据结构\""
date:       2020-03-09 01:00:00
author:     "Simon"
catalog: true
tags:
    - Data Structure
---
> “Better code, better life. ”

# 手撕数据结构---从priority_queue到heap

开始看C++ STL源码，涉及到一些标准库中的容器，这里做一下学习记录

* ### 描述

  **优先队列（priority_queue**）是STL标准库提供的一个**容器适配器（container adaptor）**，由于其底层数据结构默认是std::vector，没有真正实现一个容器，所以得以此命名。

  优先队列的本质是一个依赖于底层容器的**堆(heap)**，提供常数时间（`O(1)`）的最大元素查找，对数时间（`O(logn)`）的插入与释放。

* ### 模板形参

  ```c++
  template<class _Tp,
          class _Sequence = vector <_Tp>,
          class _Compare=std::less<typename _Sequence::value_type>>
  class priority_queue{}           
  ```

  其中：

  | 字段      | 含义                                                         |
  | --------- | ------------------------------------------------------------ |
  | _Tp       | 存储的元素类型。                                             |
  | _Sequence | 底层容器类型。容器必须满足序列容器 (SequenceContainer) 的要求，而其迭代器必须满足遗留随机访问迭代器 (LegacyRandomAccessIterator) 的要求。 |
  | _Compare  | 提供严格弱序的`Compare`类型。默认为`std::less`，**但是priority_queue会首先输出最大的元素，**因为源码实现的是`!cmp`则先出列。 |

* ### 成员

  * public member

  | 成员类型        | 定义                       |
  | --------------- | -------------------------- |
  | container_type  | Container                  |
  | value_compare   | Compare                    |
  | value_type      | Container::value_type      |
  | size_type       | Container::size_type       |
  | reference       | Container::reference       |
  | const_reference | Container::const_reference |

  * protected member

  | 成员类型     | 定义     |
  | ------------ | -------- |
  | Container c  | 底层容器 |
  | Compare comp | 比较函数 |

  * method

  | 方法名            | 说明                       |
  | ----------------- | -------------------------- |
  | priority_queue()  | 构造函数                   |
  | ~priority_queue() | 析构函数                   |
  | operator=         | assignment constructor     |
  | top               | 访问栈顶元素               |
  | empty             | 检查队列是否为空           |
  | size              | 返回容纳的元素数量         |
  | push              | 插入元素，并重新构建堆     |
  | emplace           | 原位构造元素，并重新构建堆 |
  | pop               | 删除栈顶元素               |

* ### 分析

  首先，由于成员变量的几个型别的定义依赖于`Container`，所以需要保证型别`_Tp`和`Container::value_type`一致。

  `priority_queue`主要提供三种构造方法:

  ```c++
  /* 默认构造 */
  priority_queue() : c() {}
  /* 提供自定义比较器的构造函数 */
  explicit priority_queue(const _Compare& __x) :  c(), comp(__x) {}
  /* 提供自定义底层容器的构造函数 */
  priority_queue(const _Compare& __x, const _Sequence& __s) 
      : c(__s), comp(__x) 
      { make_heap(c.begin(), c.end(), comp); }
  ```

  其中，第三种构造方法需要依据提供的容器构造一个堆。关于`make_heap`方法后面会深入探讨。

  `top`、`size`、`empty`三个方法并没有改变存储的数据，所以可以直接调用Container的相应方法，实现如下：

  ```c++
  bool empty() const { return c.empty(); }
  size_type size() const { return c.size(); }
  const_reference top() const { return c.front(); }
  ```

  `push`、`emplace`、`pop`三个方法会修改**堆**，所以需要有对应的**堆重构**操作，实现如下：

  ```c++
  void emplace(_Tp &&args) {
      c.emplace_back(args);
      push_heap(c.begin(), c.end(), comp);
  }
  
  void push(const value_type &x) {
      try {
          c.push_back(x);
          push_heap(c.begin(), c.end(), comp);
      } catch (...) {
          c.clear();
      }
  }
  
  void pop() {
      try {
          pop_heap(c.begin(), c.end(), comp);
          c.pop_back();
      } catch (...) {
          c.clear();
      }
  }
  ```

  以上是优先队列的基本接口的实现，下面来分析优先队列实现的关键--**堆**这个数据结构。

* ### 堆（heap）的概念

  堆的本质是**完全二叉树**，更确切的说是**一个可以被看做一棵树的数组对象**。堆可以分为**大顶堆（最大树）**和**小顶堆（最小树）**。**对于大顶堆，其所有父节点一定大于它的儿子节点；对于小顶堆，则每个节点必定大于它的父节点**。其在C++标准库中，heap是算法的形式提供给我们使用的。包括下面几个函数：

  - `make_heap`: 根据指定的迭代器区间以及一个可选的比较函数，来创建一个heap. O(N)
  - `push_heap`: 把指定区间的最后一个元素插入到heap中. O(logN)
  - `pop_heap`: 弹出heap顶元素, 将其放置于区间末尾. O(logN)
  - `sort_heap`：堆排序算法，通常通过反复调用`pop_heap`来实现. `N*O(logN)`

  C++11加入了两个新成员：

  - `is_heap`: 判断给定区间是否是一个heap. O(N)
  - `is_heap_until`: 找出区间中第一个不满足heap条件的位置. O(N)

  堆有两种构建方式，一种是循环插入，每次插入后的节点都保持最大堆的形式；第二种则是先把数据按顺序插入，然后从第一个叶子结点开始调整，直至满足完全二叉树的形式。

  下面来分析一下标准库里几个堆操作的源码。

* ### push_heap

  ```c++
    template<typename _RandomAccessIterator, typename _Distance, typename _Tp,
  typename _Compare>
      void
      __push_heap(_RandomAccessIterator __first,
                  _Distance __holeIndex, _Distance __topIndex, _Tp __value,
                  _Compare& __comp)
  {
      _Distance __parent = (__holeIndex - 1) / 2;
      while (__holeIndex > __topIndex && __comp(__first + __parent, __value))
      {
          *(__first + __holeIndex) = _GLIBCXX_MOVE(*(__first + __parent));
          __holeIndex = __parent;
          __parent = (__holeIndex - 1) / 2;
      }
      *(__first + __holeIndex) = _GLIBCXX_MOVE(__value);
  }
  ```
  
  先看**push_heap**这个方法，**push_heap**方法实现了插入一个新元素到堆中的功能。
  
  参数`__holeIndex`是新插入元素的下标，`__value`是新插入元素的值。
  
  函数第一行
  
  `_Distance __parent = (__holeIndex - 1) / 2;`
  
  是新插入元素的父节点，之后`while`循环体做了这么一件事：如果`__parent`节点小于插入节点，则交换它们的位置，直至`__parent`节点大于新插的节点，或者走到根结点。
  
  以上步骤，可以调整新插入的元素到正确的位置。

* ### adjust_heap

  ```c++
  template<typename _RandomAccessIterator, typename _Distance,
  typename _Tp, typename _Compare>
      void
      __adjust_heap(_RandomAccessIterator __first, _Distance __holeIndex,
                    _Distance __len, _Tp __value, _Compare __comp)
  {
      const _Distance __topIndex = __holeIndex;
      _Distance __secondChild = __holeIndex;
      while (__secondChild < (__len - 1) / 2)
      {
          __secondChild = 2 * (__secondChild + 1);
          if (__comp(__first + __secondChild,
                     __first + (__secondChild - 1)))
              __secondChild--;
          *(__first + __holeIndex) = _GLIBCXX_MOVE(*(__first + __secondChild));
          __holeIndex = __secondChild;
      }
      if ((__len & 1) == 0 && __secondChild == (__len - 2) / 2)
      {
          __secondChild = 2 * (__secondChild + 1);
          *(__first + __holeIndex) = _GLIBCXX_MOVE(*(__first
                                                     + (__secondChild - 1)));
          __holeIndex = __secondChild - 1;
      }
      __decltype(__gnu_cxx::__ops::__iter_comp_val(_GLIBCXX_MOVE(__comp)))
          __cmp(_GLIBCXX_MOVE(__comp));
      std::__push_heap(__first, __holeIndex, __topIndex,
                       _GLIBCXX_MOVE(__value), __cmp);
  }
  ```

  `adjust_heap`方法是调整一个以给定节点为根节点的子树。

  第二行：

  `_Distance __secondChild = 2 * __holeIndex + 2;`

  找到给定节点的右子节点。

  下面while循环做了这么一件事：将`__holeIndex`的两个子节点中较大的一个赋给`__holeIndex`，然后让`__holeIndex`指向较大的那个子节点，直至`__holeIndex`没有右子节点为止。

  然后将`__holeIndex`调整为最后一个元素，做一次`push_heap`。

  所以`adjust_heap`方法究竟做了什么呢？

  主要做了这么一件事：**给定一个下标__holeIndex，和一个元素__value，将__holeIndex赋值为其两个子节点中值较大的一个，重复此过程直至数组末尾，然后将新元素__value插入到以__holeIndex为根节点的子树中。**

* ### make_heap

  ```c++
  template<typename _RandomAccessIterator, typename _Compare>
  void
  __make_heap(_RandomAccessIterator __first, _RandomAccessIterator __last,
              _Compare& __comp)
  {
      typedef typename iterator_traits<_RandomAccessIterator>::value_type
          _ValueType;
      typedef typename iterator_traits<_RandomAccessIterator>::difference_type
          _DistanceType;
  
      if (__last - __first < 2)
          return;
  
      const _DistanceType __len = __last - __first;
      _DistanceType __parent = (__len - 2) / 2;
      while (true)
      {
          _ValueType __value = _GLIBCXX_MOVE(*(__first + __parent));
          std::__adjust_heap(__first, __parent, __len, _GLIBCXX_MOVE(__value),
                             __comp);
          if (__parent == 0)
              return;
          __parent--;
      }
  }
  ```

  针对元素个数大于等于2的堆，`make_heap`从最中间一个元素开始：

  `_Distance __parent = (__len - 2)/2;`

  while循环体内做了这么一件事：**深度优先遍历以`__parent `节点为根节点的子树，每次遍历的过程中都把当前父节点赋值为两个子节点中较大的一个，然后对原父节点的值做一次push_heap操作，**循环终止条件为`__parent`阶段为数组第一个元素。

* ### 代码实现

  以上三个方法是标准库里的模板函数，定义于头文件`<algorithm>`中，通用性很强，只要容器迭代器符合`_RandomAccessIterator`即可。下面尝试根据以上三种方法实现一个堆排序：

  ```c++
  //
  // Created by Simon on 2020/3/8.
  //
  #include <iostream>
  #include <vector>
  
  using namespace std;
  
  void push_heap(int *p, int holeIndex, int topIndex, int value) {
      int parent = (holeIndex - 1) / 2;
      while (holeIndex > topIndex && *(p + parent) < value) {
          *(p + holeIndex) = *(p + parent);
          holeIndex = parent;
          parent = (holeIndex - 1) / 2;
      }
      *(p + holeIndex) = value;
  }
  
  void adjust_heap(int *p, int holeIndex, int len) {
      int secondChild = holeIndex;
      const int topIndex = holeIndex;
      int value = *(p + holeIndex);
      while (secondChild < (len - 1) / 2) {
          secondChild = secondChild * 2 + 2;
          if (*(p + secondChild) < *(p + secondChild - 1)) secondChild--;
          *(p + holeIndex) = *(p + secondChild);
          holeIndex = secondChild;
      }
  
      if ((len & 1) == 0 && len == (len - 2) / 2) {
          secondChild = secondChild * 2 + 2;
          *(p + holeIndex) = *(p + secondChild-1);
          holeIndex = secondChild-1;
      }
      push_heap(p, holeIndex, topIndex, value);
  }
  
  void make_heap(int *p, int len) {
      if (len < 2)
          return;
      int parent = (len - 2) / 2;
      while (parent >= 0) {
          adjust_heap(p, parent, len);
          parent--;
      }
  }
  
  int main() {
  
      int a[10] = {5, 4, 1, 2, 7, 8, 22, 13, 41, 9};
      make_heap(a, 10);
      std::vector<int> c;
      for (int &i:a) {
          cout << i << " ";
      }
      cout << endl;
      return 0;
  }
  ```

  Output：

  ```c++
  41 13 22 5 7 8 1 4 2 9
  ```

  一个大顶堆！





