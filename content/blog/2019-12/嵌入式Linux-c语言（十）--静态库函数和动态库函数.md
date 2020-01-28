---
title : "嵌入式Linux-C语言（十）--静态库函数和动态库函数"
date : "2019-12-24T10:13:18+08:00"
tags : ["静态函数","动态函数","C"]
series : ["嵌入式Linux-C语言"]
categories : ["编程技术"]
draft : false
toc : true
---

## 一、静态链接库

静态链接库是obj文件的一个集合，通常静态链接库以".a"为后缀，名字格式一般为libxxx.a，由程序ar生成。静态链接库是在程序编译过程中链接的，已经将调用的相关函数拷贝到程序内部，程序运行时和静态链接库已经没有任何关系。
<!--more-->
### 1、静态链接库的创建

**A、编写源码库文件**

源码库文件一般包含.c和.h文件，

hello.c文件：

```
#include <stdio.h>

void display(void)
{
    printf("hello world\n");
}
```

hello.h文件：

```
#ifndef __HELLO_H
#define __HELLO_H

void display(void);

#endif
```

**B、编译源码库文件**

```
gcc -o hello.o -c hello.c
```

生成hello.o目标文件

**C、将目标文件归档生成静态链接库文件**

```
ar -cr libhello.a hello.o
```

**D、发布静态链接库**

一般来说，静态链接库需要发布libxxx.a和.h文件，.h文件可以让第三方开发者了解静态链接库中的各函数的功能和函数声明，libxxx.a文件是第三方开发者在调用静态链接库中的函数后在编译链接阶段链接的库。

### 2、静态链接库的使用
**A、查阅静态链接库的.h文件**

获取发布的静态链接库后，查看.h文件，看静态链接库的各个函数功能和函数声明。

**B、使用静态链接库的某个函数**

使用静态链接库时需要声明静态链接库的.h文件

```
#include "hello.h"

int main(int argc, char**argv)
{
    display();
    return 0;
}
```

### C、编译工程文件

编译工程文件时，需要在编译链接时添加相关选项：

> -Lpath：表示在path目录中搜索库文件，如-L.则表示在当前目录。
> 
> -lxxx:表示要链接的静态链接库为libxxx.a
> 
> -static:表示将所有链接的库静态加载
> 
> gcc -o main main.c -L. -lhello

## 二、动态链接库
    
动态链接库是程序运行时加载的库，当动态链接库正确安装后，所有的程序都可以使用动态库来运行程序。动态链接库是目标文件的集合，目标文件在动态链接库中的组织方式是按照特殊方式形成的。库中函数和变量的地址是相对地址，不是绝对地址，其真实地址在调用动态库的程序加载时形成。

### 1、动态链接库的创建
**A、编写源码库文件**

源码库文件一般包含.c和.h文件，

hello.c文件：

```
#include <stdio.h>
void display(void)
{
    printf("hello world\n");
}
```

hello.h文件：

```
#ifndef __HELLO_H
#define __HELLO_H

void display(void);

#endif
```
**B、编译源码库文件**

```
gcc -fPIC -c hello.c -o hello.o
```

生成目标文件hello.o

-fPIC选项的作用是使得gcc生成的代码是位置无关的

### C、生成动态链接库

```
gcc -shared -o libhello.so hello.o
```

生成动态链接库libhello.so文件

-shared选项告诉编译器生成一个动态链接库

### 2、动态链接库的使用

**A、查阅动态链接库的.h文件**

获取发布的动态链接库后，查看.h文件，看动态链接库的各个函数功能和函数声明。

**B、使用动态链接库的某个函数**

使用动态链接库时需要声明动态链接库的.h文件

```
#include "hello.h"

int main(int argc, char**argv)
{
    display();
    return 0;
}
```

**C、编译工程文件**

编译工程文件时，需要在编译链接时添加相关选项：

> -Lpath：表示在path目录中搜索库文件，如-L.则表示在当前目录。
> 
> -lxxx:表示要链接的动态链接库为libxxx.so
> 
>  gcc -o main main.c -L. -lhello

**D、将动态链接库文件注册到系统环境变量中的库加载路径**

方法一：将动态链接库文件拷贝到系统环境变量中的库加载路径中的某个目录

cp libhello.so /usr/lib

方法二：将当前目录添加为统环境变量中的库加载路径

把当前工作目录加入动态链接库的搜索路径配置文件/etc/ld.so.conf中。

 
如果没有以上操作，运行时程序将会报错：


```
error while loading shared libraries: libhello.so: cannot open shared object file: No such file or directory
```

程序运行时将会到相应目录下加载动态链接库中的函数执行。

**E、程序运行时的库依赖**

ldd命令可以查询程序运行时需要的依赖库


```
ldd main

linux-vdso.so.1 =>  (0x00007fff265d8000)

libhello.so => /usr/lib/libhello.so (0x00007f15d8af1000)

libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6 (0x00007f15d8733000)

/lib64/ld-linux-x86-64.so.2 (0x00007f15d8d05000)
```
