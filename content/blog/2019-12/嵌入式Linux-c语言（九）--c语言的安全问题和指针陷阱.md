---
title : "嵌入式Linux-C语言（九）--C语言的安全问题和指针陷阱"
date : "2019-12-24T10:10:44+08:00"
tags : ["指针","安全问题","C"]
series : ["嵌入式Linux-C语言"]
categories : ["编程技术"]
draft : false
toc : true
---

C语言是灵活度和自由度较大的编程语言，作为C语言核心的指针更是让C语言程序员可以越过安全的栅栏，对某些内存区域进行破坏性访问，引发安全风险。很多安全问题都能追根溯源到指针的误用。本文将从指针的角度解读C语言常见的安全问题和指针陷阱。
<!--more-->
## 一、指针的声明和初始化
### 1、不恰当的指针声明

```
int* ptr1, ptr2;//声明ptr1为int指针，ptr2为整型

int *ptr1, *ptr2;//ptr1,ptr2都声明为指针

#define PINT int *

PINT ptr1, ptr2;//等价于int* ptr1, ptr2;
```

推荐方式：

```
typedef int * PINT

PINT ptr1, ptr2;//等价于int *ptr1, *ptr2;
```

### 2、使用指针前未初始化

初始化指针前使用指针会导致运行时错误，这种指针称为野指针。

```
int *p;
printf(“%d\n”, *p);
```

指针变量p未被赋值指针（地址），此时*p将会导致不可预知的情况。

### 3、处理未初始化指针

有三种方法可以用来处理未初始化的指针：

**A、用NULL初始化指针**

```
int *p = NULL;
if(NULL == p)
{

}
```
**B、用assert函数**
```
assert(NULL != p);
```
**C、用第三方工具**



## 二、误用指针

很多安全问题聚焦于缓冲区溢出，覆写对象边界以外的内存就会导致缓冲区溢出，这块内存可能是本程序的地址空间，也可能是其他进程的，如果是程序地址空间以外的内存，大部分操作系统会发出一段错误然后中止程序。如果缓冲区溢出发生在应用程序的地址空间内，就会导致对数据的未授权访问和控制转移到其他代码段，导致系统被攻陷。下列情况可能导致缓冲区溢出：

- **A、访问数组元数时没有检查索引项**
- **B、对数组指针做指针算数运算时不够小心**
- **C、用gets这样的函数从标准输入读取字符串**
- **D、误用strcpy和strcat这样的函数**

如果缓冲区溢出发生在栈帧的元数上，就可能把栈帧的返回地址部分覆写为对同一时间创建的恶意代码的调用。

### 1、测试NULL

对于动态分配内存函数一定要检查返回值，否则如果内存分配失败程序可能会非正常终止。

```
char *vetcor = (char *)malloc(128*sizeof(char));
if(NULL == vector)
{
//Malloc failure.
}
```

### 2、错误使用解引操作

声明和初始化指针的常用方法如下：

```
int num;
int *p = num;
```

但是以下是错误的

```
int num;
int *p ;
*p = num;//正确为p = &num
```

### 3、迷途指针

释放指针后却仍然在引用原来的内存，就会产生迷途指针。如果在释放指针后仍然在试图操作原来的内存，读操作可能会返回无效数据，写操作可能会破坏这块内存的值，导致其他正在使用这块内存的程序出现异常。

### 4、数组访问越界

C语言中数组并没有提供防止访问数组越界的机制，因此必须由程序员保证对数组的访问不越界。尤其是用指针方式访问数组元素时一定要保证不能越过数组边界。

### 5、错误计算数组长度

将数组传递给函数时，一定要同时传递数组长度。数组长度参数可以避免缓冲区溢出。==strcpy函数就是一个允许缓冲区溢出的函数，因此尽量避免使用，使用具有缓冲区保护机制的strncpy函数==。

## 三、释放问题
### 1、重复释放

重复释放是指将同一块内存释放两次，如：


```
char *name = (char *)malloc(....);

...

free(name);

...

free(name);
```

为了避免对同一块内存的重复释放，一般释放指针后将指针置为NULL。


```
char *name = (char *)malloc(....);

...

free(name);

name = NULL;
```

### 2、清除敏感信息

当应用程序终止后，大部分操作系统都不会把用到的内存清零或执行别的操作，系统可能会将之前用过的内存分配给其他程序使用，如果这些内存原来保存的是身份信息、密码信息等敏感数据，这样做就是不安全的，因为其他人可以使用这部分内存，可以通过覆写将内存中敏感数据清空。


## 四、指针类型转换

指针类型的转换可以实现一些特殊的功能，如：

**访问有特殊目的的地址**

**判断机器的字节序**

### 1、访问特殊用途的地址

在嵌入式系统开发中，很多特殊功能寄存器是统一编址的，通过访问这些特殊功能寄存器可以控制相应的外设的功能。


```
#define WTCON ( *((unsigned long *)0xE2700000))

WTCON |= 0X3<<8;
```


通过指针类型的转换，可以WTCON寄存器的某些位设置为1，进而控制外设。

### 2、判断机器的字节序

字节序是指数据在内存单元中字节的存储顺序。字节序一般分为小字节序和大字节序，也称小端模式和大端模式。小端模式表示整数的4字节的中的低地址存储整数数据的低位。通过将整数的地址从指针转换为char，打印出每个字节的内存就可以知道机器的字节序。


```
#include <stdio.h>

int main(int argc, char**argv)
{

    int num = 0x12345678;
    char *p = (char *)num;
    int i;
    for(i = 0; i < 4; i++)
    {
        printf("%p:%2x\n", p,(unsigned char)*p++);
    }
    return 0;
}
```


运行结果如下：

0x7fffe8f13cb9:78

0x7fffe8f13cba:56

0x7fffe8f13cbb:34

0x7fffe8f13cbc:12

结论：低位存储低字节，小端模式。