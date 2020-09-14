---
layout:     post
title:      "结构体sockaddr、sockaddr_in、sockaddr_in6探究"
subtitle:   "\"跨平台网络库编写\""
date:       2020-09-10 20:00:00
author:     "Simon"
catalog: true
header-img: "img/se-6.jpg"
tags:
   - C++
---

> “Better code, better life. ”

## 结构体sockaddr、sockaddr_in、sockaddr_in6探究

最近在写一个跨平台网络库，发现一些网络编程的系统调用的参数在不同平台上各不相同，令人头大。今天把sockaddr、sockaddr_in、sockaddr_in6这三个结构体放到一起，从源码的层面做一次研究。

### Linux环境

话不多说，直接贴源码：

#### sockaddr

```c++
/* /usr/include/bits/socket.h */
/* Structure describing a generic socket address.  */
struct sockaddr
{
 __SOCKADDR_COMMON (sa_);    /* Common data: address family and length.  */
 char sa_data[14];           /* Address data.  */
};

```

其中宏`__SOCKADDR_COMMON`的定义在`usr/inlcue/bits/sockaddr.h`中：

```c
/*usr/inlcue/bits/sockaddr.h*/
/* POSIX.1g specifies this type name for the `sa_family' member.  */
typedef unsigned short int sa_family_t;

/* This macro is used to declare the initial common members
of the data types used for socket addresses, `struct sockaddr',
`struct sockaddr_in', `struct sockaddr_un', etc.  */

#define __SOCKADDR_COMMON(sa_prefix) sa_family_t sa_prefix##family

#define __SOCKADDR_COMMON_SIZE  (sizeof (unsigned short int))
```

现在我们能看出，这三个结构的第一个字段都是一个`unsigned short int` 类型，只不过用宏来定义了三个不同的名字。

#### sockaddr_in

```c
/* /usr/include/netinet/in.h */
/* Structure describing an Internet socket address.  */
struct sockaddr_in
{
 __SOCKADDR_COMMON (sin_);
 in_port_t sin_port;         /* Port number.  */
 struct in_addr sin_addr;    /* Internet address.  */

 /* Pad to size of `struct sockaddr'.  */
 unsigned char sin_zero[sizeof (struct sockaddr) -
            __SOCKADDR_COMMON_SIZE -
            sizeof (in_port_t) -
            sizeof (struct in_addr)];
};
```

第二个结构`sockaddr_in`多了两个成员，`in_port_t`和`struct in_addr`，它们定义在`/usr/include/netinet/in.h`中：

```c
/* Type to represent a port.  */
typedef uint16_t in_port_t;

/* Internet address.  */
typedef uint32_t in_addr_t;
struct in_addr
{
 in_addr_t s_addr;
};
```

这么看来`sockaddr_in`这个结构也不复杂，除了一开始的2个字节表示`sin_family`，然后是2个字节的变量`sin_port`表示端口，接着是4个字节的变量`sin_addr`表示IP地址，最后是8个字节变量`sin_zero`填充尾部，用来与结构`sockaddr`对齐。

#### sockaddr_in6

```c
/* /usr/include/netinet/in.h */

#ifndef __USE_KERNEL_IPV6_DEFS

/* Ditto, for IPv6.  */
struct sockaddr_in6
{
 __SOCKADDR_COMMON (sin6_);
 in_port_t sin6_port;        /* Transport layer port # */
 uint32_t sin6_flowinfo;     /* IPv6 flow information */
 struct in6_addr sin6_addr;  /* IPv6 address */
 uint32_t sin6_scope_id;     /* IPv6 scope-id */
};
#endif /* !__USE_KERNEL_IPV6_DEFS */
```

`sockaddr_in6`又多了一个未知的结构`in6_addr`，经过寻找发现其定义也在`/usr/include/netinet/in.h`中

```c
#ifndef __USE_KERNEL_IPV6_DEFS

/* IPv6 address */
struct in6_addr
{
 union
 {
     uint8_t __u6_addr8[16];
#if defined __USE_MISC || defined __USE_GNU
     uint16_t __u6_addr16[8];
     uint32_t __u6_addr32[4];
#endif
 } __in6_u;
    
#define s6_addr         __in6_u.__u6_addr8
#if defined __USE_MISC || defined __USE_GNU
# define s6_addr16      __in6_u.__u6_addr16
# define s6_addr32      __in6_u.__u6_addr32

#endif

};

#endif /* !__USE_KERNEL_IPV6_DEFS */
```

这个结构看起来有点乱，但是如果抛开其中的预编译选项，其实就是8个字节，用来表示IPV6版本的IP地址，一共128位，只不过划分字节的段数有些不同，每段字节多一点那么段数就少一点，反义亦然。

#### 总结

那接下来我们整理一下，为了看的清楚，部分结构使用伪代码，不能通过编译，主要是方便对比，整理如下

```
/* Structure describing a generic socket address.  */
struct sockaddr
{
 uint16 sa_family;           /* Common data: address family and length.  */
 char sa_data[14];           /* Address data.  */
};

/* Structure describing an Internet socket address.  */
struct sockaddr_in
{
 uint16 sin_family;          /* Address family AF_INET */ 
 uint16 sin_port;            /* Port number.  */
 uint32 sin_addr.s_addr;     /* Internet address.  */
 unsigned char sin_zero[8];  /* Pad to size of `struct sockaddr'.  */
};

/* Ditto, for IPv6.  */
struct sockaddr_in6
{
 uint16 sin6_family;         /* Address family AF_INET6 */
 uint16 sin6_port;           /* Transport layer port # */
 uint32 sin6_flowinfo;       /* IPv6 flow information */
 uint8  sin6_addr[16];       /* IPv6 address */
 uint32 sin6_scope_id;       /* IPv6 scope-id */
};
```

#### 强转的可能性

现在，我们可以知道结构 `sockaddr` 和 `sockaddr_in` 字节数完全相同，都是16个字节，所以可以直接强转，但是结构 `sockaddr_in6` 有28个字节，为什么在使用的时候也是直接将地址强制转化成(sockaddr*)类型呢？

其实`sockaddr` 和 `sockaddr_in` 之间的转化很容易理解，因为他们开头一样，内存大小也一样，但是`sockaddr`和`sockaddr_in6`之间的转换就有点让人搞不懂了，其实你有可能被结构所占的内存迷惑了，这几个结构在作为参数时基本上都是以指针的形式传入的，我们拿函数`bind()`为例，这个函数一共接收三个参数，第一个为监听的文件描述符，第二个参数是`sockaddr*`类型，第三个参数是传入指针原结构的内存大小，所以有了后两个信息，无所谓原结构怎么变化，因为他们的头都是一样的，也就是`uint16 sa_family`，那么我们也能根据这个头做处理，原本我没有看过`bind()`函数的源代码，但是可以猜一下:

```c
int bind(int socket_fd, sockaddr* p_addr, int add_size)
{
    if (p_addr->sa_family == AF_INET)
    {
        sockaddr_in* p_addr_in = (sockaddr_in*)p_addr;
        //...
    }
    else if (p_addr->sa_family == AF_INET6)
    {
        sockaddr_in6* p_addr_in = (sockaddr_in6*)p_addr;
        //...
    }
    else
    {
        //...
    }
}
```

由以上代码完全可以实现IPv4和IPv6的版本区分，所以不需要纠结内存大小的不同。

### Windows环境

windows环境下只有`struct sockaddr`、`sockaddr_in`两种结构，都定义在`_ip_types.h`中，源码为：

```c
struct sockaddr {
	u_short	sa_family;
	char	sa_data[14];
};

struct sockaddr_in {
	short	sin_family;
	u_short	sin_port;
	struct in_addr	sin_addr;
	char	sin_zero[8];
};
```

其中`in_addr`结构定义在`inadr.h`中：

```c
typedef struct in_addr {
  union {
    struct { u_char  s_b1, s_b2, s_b3, s_b4; } S_un_b;
    struct { u_short s_w1, s_w2; } S_un_w;
    u_long S_addr;
  } S_un;
} IN_ADDR, *PIN_ADDR, *LPIN_ADDR;
```

 `struct in_addr`结构体中，使用联合union，三种方式来保存IP地址信息；

关于IP地址，这是一个4字节的无符号整型，此结构体也就对应了三种保存方式：

1. S_un_b, 单字节保存为；
2. S_un_w,双字节保存；
3. S_addr,4字节保存；

我们常用S_addr4字节直接保存IP地址信息。

最后是`unsigned char sin_zero[len]`用来充填对齐，使`sockaddr_in`与`sockaddr`内存对齐。

#### sockaddr_in6

`sockaddr_in6`定义在头文件`ws2tcpip.h`里，结构为：

```c
#  define __C89_NAMELESS __MINGW_EXTENSION

struct sockaddr_in6 {
  short sin6_family;
  u_short sin6_port;
  u_long sin6_flowinfo;
  struct in6_addr sin6_addr;
  __C89_NAMELESS union {
    u_long sin6_scope_id;
    SCOPE_ID sin6_scope_struct;
  };
};
```

其中`SCOPE_ID`定义在`ws2def.h`，结构为：

```c
typedef struct _SCOPE_ID {
  __C89_NAMELESS union {
    __C89_NAMELESS struct {
	ULONG	Zone : 28;
	ULONG	Level : 4;
    };
    ULONG Value;
  };
} SCOPE_ID, *PSCOPE_ID;
```

`in6_addr`定义在头文件`in6addr.h`中，结构为：

```c
typedef struct in6_addr {
  union {
    u_char Byte[16];
    u_short Word[8];
#ifdef __INSIDE_CYGWIN__
    uint32_t __s6_addr32[4];
#endif
  } u;
} IN6_ADDR, *PIN6_ADDR, *LPIN6_ADDR;
```

看起来和Linux下大同小异。其成员变量含义为：

* **sin6_family**，协议族，必须为AF_INET6
* **sin6_port**，端口号
* **sin6_flowinfo**，ipv6流信息
* **sin6_addr**，地址
* **sin6_scope_id**，接口集

