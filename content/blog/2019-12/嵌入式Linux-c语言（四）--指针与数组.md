---
title : "嵌入式Linux-C语言（四）--指针与数组"
date : "2019-12-24T09:47:47+08:00"
tags : ["指针","数组","C"]
series : ["嵌入式Linux-C语言"]
categories : ["编程技术"]
draft : false
toc : true
---

数组是C语言内建的数据结构，彻底理解数组及其用法是开发高效应用程序的基础。数组和指针紧密关联，但又不是完全可以互换。
<!--more-->
## 一、数组简介

数组是能用索引访问的同种类型元素的连续集合。数组的元素在内存中是相邻的，中间不存在空隙，数组的元素是相同类型的。

### 1、数组的解读
    
数组的定义：int a[10] = {0,1,2,3,4,5};

a[0]:数组的第一个元素，首元素（做左值时表示第0个元素的内存空间）

&a:数组的地址，是常量，不能做左值，类型等同int (*)[10]（数组指针）。

&a[0]:数组第0个元素的地址，与数组名a等价

a:a是数组名，不能做左值，做右值时表示数组首元素的地址，与&a[0]相同。

数组的地址与数组首元素的地址不是一个概念。

数组名可以看作const指针，但数组名作为sizeof操作符的参数和&运算符的参数除外，数组名不是指针。

数组本质是一块连续的内存空间。

### 2、一维数组

一维数组是线性结构，用一个索引访问成员。

    int vector[5] = {1,2,3,4,5};

数组的内部表示不包含其元数数量的信息，数组名字只是引用了一块内存。

### 3、二维数组
    
二维数组使用行和列来标识数组元素，二维数组可以看作是数组的数组。

    int vetor[2][3] = {{1,2,3},{4,5,6}};

二维数组的数组名等价于数组指针int (*vector)[3];

## 二、数组和指针表示法

### 1、数组元数的访问方式

A、数组下标访问

    数组名[索引下标]；

    a[i] <==>i[a]

B、指针方式访问

    *（指针+偏移量）；

    *(a + 2);//等价于a[2]

    数组中的元数地址是连续的。

C、数组名的指针运算

a + n ==> p + n*sizeof(*a)

&a + 1 ==> &a + sizeof(a)

C语言没有强制规定数组的边界，因此用无效的索引访问数组将会造成数组访问越界，造成不可预期的行为。

### 2、数组与指针的区别
   
int vector[5] = {1,2,3,4,5};

int *p = vector;

vector[i]访问数组元素的方式表示从地址vector开始，移动i个位置取出内容

*(vetcor+i)访问数组元素方式表示从vector开始，在地址上增加i，取出地址中的内容。

sizeof(vector)得到的是数组分配的字节数

sizeof(p)得到的是指针变量p的长度

## 三、数组作为函数参数

### 1、传递一维数组
将一维数组作为参数传递给函数实际上是通过值来传递数组的地址，不需要传递整个数组，不用再栈上分配空间，通常也需要传递数组长度，确保对数组的访问不会越界。

在函数声明中声明数组的方法有两种：

**A、数组表示法**

    void display(int a[], int size);

Ps：通过使用sizeof(a)计算数组的元数数量可能造成错误，sizeof(a)/sizeof(int)是正确的。

**B、指针表示法**

    void display(int *a, int size);

数组作为函数参数时，数组会退化为指针。

### 2、传递多维数组
传递多维数组时，需要在函数原型声明中确定用数组表示法还是指针表示法，以及数组的维数和每一维的大小。要想在函数内部使用数组表示法，必须指定数组的形态，否则编译器将无法使用下标。

```
void display(int a[][5], int rows);

void display(int (*a)[5], int rows);

void display(int *a[5], int rows);
//错误声明，语法没错，但是编译器会认为函数传入的数组有5个整型指针。

```

传递二维以上数组时，除了第一维以外，需要指定其他维度的长度。

二维数组作为函数参数时，二维数组退化为数组指针。

指针数组作为函数参数时，指针数组退化为二维指针。

## 四、数组指针

### 1、数组类型

C语言中数组有自己特定的类型，数组的类型由数组元数和数组大小决定。

```
int array[n]的数组类型为int[n]；
```

C语言中可以使使用typedef对数组类型重命名。

```
typedef    type(name)[size];name为数组类型
```

代码实例如下：

```
#include <stdio.h>

typedef int (array)[5];

int main(int argc, char *argv[])
{
    array a;
    int i;
    for(i = 0; i < sizeof(a) / sizeof(int); i++)
    {   
        a[i] = i;
        printf("%d\n", a[i]);
    }   
    return 0;
}
```

### 2、数组指针

数组指针是指向数组的指针。

```
int (*p)[5];//数组指针
```

数组指针是指向数组的指针变量，数组指针持有的是数组的地址，相当于一个二级指针。

```
int a[5];

p = &a;//&a是数组的地址，与数组指针类型相同
```


数组指针可以通过数组类型定义，也可以直接定义。

数组类型的指针定义：

```
typedef int (array)[5];

array *a;
```

数组指针的直接定义：

```
type(*array)[n];
```

数组指针类型的定义：

```
typedef    type(*ArrayPointer)[n];
```

数组指针的运算：

array + 1 ===> array + sizeof(*array) ==>array + sizeof(int[5])

代码实例：


```
#include <stdio.h>

typedef int (*ArrayPointer)[5];

int main(int argc, char *argv[])
{
    ArrayPointer pa;
    int (*p)[5];
    int a[5] = {1,2,3,4,5};
    pa = &a;
    p = &a;
    int i;

    for(i = 0; i < sizeof(a) / sizeof(int); i++)
    {   
        printf("%d\n", a[i]);
        (*pa)[i] = 10;
        printf("%d\n", a[i]);
        (*p)[i] = i+1;
        printf("%d\n", a[i]);
    }   
    return 0;
}
```


### 3、指针数组


指针数组是元素类型为指针的数组，指针数组的定义如下：

type * array[n];

数组array中有n个元素，每个元素存储type *指针。

int *vector[5];  //指针数组，数组中元素的是指针变量

代码实例：


```
#include <stdio.h>

int main(int argc, char *argv[])
{
    const char *keyword[] =
    {   
        "do",
        "for",
        "return",
        "while",
        NULL
    };  

    unsigned int i = 0;
    while(keyword[i] != NULL)
    {   
        printf("%s\n", keyword[i++]);
    }   
    return 0;
}
```

指针数组与数组指针的内存分配如下图：

![4-1](/images/blog/C/4-1.png)
