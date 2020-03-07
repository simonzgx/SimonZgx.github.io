---
layout:     post
title:      "工作问题记录--python str类型踩坑记"
subtitle:   " \"Python\""
date:       2020-03-05 22:00:00
author:     "Simon"
catalog: true
tags:
    - Python
---
> “Better code, better life. ”

# python string类型踩坑记

非专业`python`程序员小张今天用`python`写了个脚本，不出所料又出岔子了→.→

* ### 问题描述

  问题起源两个变量的对比

  ```python
  str1 = b'abcd'
  str2 = 'abcd'
  ```

  str1类型是bytes，str2类型是string，之前写`golang`对于[]byte和string类型基本可以等同对待，所以我天真的以为`python` string的底层是bytes，于是写下了这行代码

  ```python
  if str(str1) == str2 :
  	#do something
  ```

  显然我真的太天真了。。。

* ### 问题分析

  先来看看`golang`类似情况的处理

  ```go
  var bf bytes.Buffer
  bf.WriteByte('a')
  var b []byte
  b = append(b, 'a')
  var str string
  str = "a"
  fmt.Println(str == bf.String())
  fmt.Println(str == string(b))
  fmt.Println(string(b) == bf.String())
  ```

  Output:

  ```go
  true
  true
  true
  ```

  **后来我去了解了下，`golang`里的string也不是简单的等于[]byte，这里不做深入讨论**

  对于python2[官方文档](https://docs.python.org/2/reference/lexical_analysis.html#string-literals)对string类型有如下说明：

  ```
* The backslash (\) character is used to escape characters that otherwise have a special meaning, such as newline, backslash itself, or the quote character.
  * String literals may optionally be prefixed with a letter 'r' or 'R'; such strings are called raw strings and use different rules for interpreting backslash escape sequences. A prefix of 'u' or 'U' makes the string a Unicode string. 
  * A prefix of 'b' or 'B' is ignored in Python 2; it indicates that the literal should become a bytes literal in Python 3 (e.g. when code is automatically converted with 2to3). A 'u' or 'b' prefix may be followed by an 'r' prefix.
  ```
  
  `python2`中，除了`b`以外，字符串的`prefix`还包括`r` \ `R` ，`u`  \ `U` 来分别标识该字符串是`raw string` 和`unicode string` 。而`b` 在`python2`中是被忽略的。

  **python3中是这么说的：**
  
  ```
  * Bytes literals are always prefixed with 'b' or 'B'; they produce an instance of the bytes type instead of the str type. They may only contain ASCII characters; bytes with a numeric value of 128 or greater must be expressed with escapes.
  ```
  
  1. 以`b`开头的是型别是**字节数组**
  2. 一个字节只有8个bit，所以`Bytes`只包括ASCII码 
  
  同样的**c++中std::string底层的数据结构是char*，而char类型占2个字节**
  
  所以我们得到一个结论：**A CHARACTER IS NOT A BYTE**
  
* ### 总结

  我们用`string`来输出**文本类型** ，比如:

  ```python
  print('שלום עולם')
  ```

  Output:

  ```
  שלום עולם
  ```

  我们用`bytes`来输出更底层的信息，比如上面的字符串在计算机中是如何用`01`存储的：

  ```python
  bytes('שלום עולם', 'utf-8')
  ```

  Output:

  ```
  b'\xd7\xa9\xd7\x9c\xd7\x95\xd7\x9d \xd7\xa2\xd7\x95\xd7\x9c\xd7\x9d'
  ```

  但是`bytes`和`str`之间的转换一定要加`encode`和`decode`的，我上面就是犯了这么一个愚蠢的错误，以下几段代码很能说明问题

  ```python
  b'\xE2\x82\xAC'.decode('UTF-8')
  ```

  Output:

  ```python
  '€'
  ```

  但是不能直接做`append`操作，因为不存在从`bytes`到`str`的隐式转换

  ```python
  b'\xEF\xBB\xBF' + 'Text with a UTF-8 BOM'
  ```

  Output:

  ```python
  Traceback (most recent call last):
    File "<stdin>", line 1, in <module>
  TypeError: can't concat bytes to str
  ```

  由于`A`的ASCII码是`41`所以这两种写法是

  ```python
  b'A' == b'\x41'
  ```

  Output:

  ```python
  True
  ```

  但是

  ```python
  'A' == b'A'
  ```

  Output:

  ```python
  False
  ```

  





