---
title : "嵌入式Linux-C语言（五）--指针与字符串"
date : "2019-12-24T09:58:20+08:00"
tags : ["指针","字符串","C"]
series : ["嵌入式Linux-C语言"]
categories : ["编程技术"]
draft : false
toc : true
---

## 一、字符串简介
### 1、字符串声明
声明字符串的方式有三种：字面量、字符数组、字符指针。
<!--more-->

字符串字面量是用双引号引起来的字符序列，常用来进行初始化，位于字符串字面量池中，字符字面量是用单引号引起来的字符。

字符串字面量池是程序分配的一块内存区域，用来保存组成字符串的字符序列。多次用到一个字符串字面量时，字符串字面量池中通常只保存一份副本，一般来说字符串字面量分配在只读内存中，是不可变的，但是当把编译器有关字面量池的选项关闭时，字符串字面量可能生成多个副本，每个副本拥有自己的地址。

GCC编译器中字符串字面量是可以改变的，为了将字符串指针变量声明为常量可以用const修饰。

字符串是以ASCII字符NUL结尾的字符序列。字符串通常存储在数组或者从堆上分配的内存中。并非所有的字符数组都是字符串，字符数组可能没有NUL字符。

字符串的长度是字符串中除了NULL字符之外的字符数。

字符常量是单引号引起来的字符序列，通常由一个字符组成。字符的长度是1个字节，字符字面量的长度是4个字节。sizeof（char）= 1，sizeof（‘a’） =  4。

字符数组是一个数组，每个元素的值都可以改变。而字符串指针指向的是一个常量字符串，它被存放在程序的静态数据区，一旦定义就不能改变。这是最重要的区别。

### 2、字符串初始化
初始化字符串采用的方法取决于变量是被声明位字符数组还是字符指针，为字符串分配的内存要么是数组要么是指针指向的一块内存。

初始化字符数组：


```
char buffer[] = “hello world”;//字符串的长度为11，字面量需要12个字节

char buffer[12];

strcpy(buffer, “hello world”);
```

初始化字符指针：


```
char *buffer = (char *)malloc(strlen(“hello world”) + 1);

strcpy(buffer, “hello world”);
```


## 二、传递字符串
函数中经常将参数声明为字符指针。

### 1、传递简单字符串
直接传递字符串字面量：


```
char buffer[12];

strcpy(buffer, “hello world”);
```


传递字符数组：


```
char src[] = “hello world”;//字符串的长度为11，字面量需要12个字节

char dest[12];

strcpy(dest, src);
```

 

为了避免传入的字符串被修改，可以将传递的形参声明为const。


```
const char src[] = “hello world”;//字符串的长度为11，字面量需要12个字节

char dest[12];

strcpy(dest, src);//char *strcpy(char *dest, const char *src);
```


### 2、传递需要初始化的字符串
函数返回需要函数初始化的字符串，因此需要向函数传递缓冲区。

A、必须传递缓冲区的地址和长度

B、调用者负责释放缓冲区

C、函数通常返回缓冲区的指针


```
char pfun(char *buffer, int size)
{
    xxxxx;
    return buffer;
}
```


### 3、给应用程序传递参数

```
int main(int argc, char **argv)

{

}
```


通过应用程序的入口函数main可以像应用程序传递参数

## 三、返回字符串
函数返回字符串时返回的实际是字符串的地址，返回的地址必须是合法的。

### 1、返回字面量的地址

```
Char *returnstring(int code)
{

    xxx;

    return “hello world”;
}
```


### 2、返回动态分配内存的地址
在函数内从堆上动态分配字符串的内存，返回地址。


```
char *blanks(int num)
{
    char *buffer = (char *)malloc(num + 1);
    strcpy(buffer, “hello world”);
    xxx;
    retrurn buffer;
}
```

使用完成后，函数调用者必须释放内存，否则会造成内存泄漏。

### 3、返回局部字符串的地址
返回局部字符串的地址是不可取的，局部字符串所在的内存会被其他栈帧覆盖。


```
char *blanks(void)
{
    char buffer[] = "hello world";
    return buffer;
}
```
