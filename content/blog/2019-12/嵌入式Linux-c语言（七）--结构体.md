---
title : "嵌入式Linux-C语言（七）--结构体"
date : "2019-12-24T10:05:28+08:00"
tags : ["结构体","C"]
series : ["嵌入式Linux-C语言"]
categories : ["编程技术"]
draft : false
toc : true
---

<!--more-->
## 一、结构体简介
### 1、结构体定义
结构体定义一般有两种方法较为常用：

**第一种方法：**

```
struct person{
char *name;
unisgned int age;
};
```

**第二种方法：**

```
typedef struct person{
char *name;
unsigned int age;
}Person;
```

**person实例声明如下：**

```
Person person;//声明一个person对象

Person *ptrPerson =(Person*)malloc(sizeof(Person));//声明一个person对象，并分配内存
```


### 2、结构体的初始化
使用结构体声明的结构体对象的初始化：


```
Person person;

person.name = (char *)malloc(strlen(“xiaoming”) + 1);

strcpy(person.name, “xiaoming”);

person.age = 30;
```


使用结构体指针声明的结构体的初始化：


```
Person *ptrPerson;

prtPerson = (Person *)malloc(sizeof(Person));

ptrPerson->name = (char *)malloc(strlen(“xiaoming”) + 1);

strcpy(ptrPerson->name, “xiaoming”);

prtPerson->age = 30;
```

### 3、结构体释放的问题
定义结构体时会为结构体分配内存，但运行时系统不会自动为结构体内部的指针分配内存，结构体销毁时，运行时系统也不会自动释放结构体内部的指针指向的内存。结构体内部的指针指向内存一般为动态分配的内存，使用完成后需要释放，否则将会造成内存泄漏。

### 4、动态分配内存开销的避免
重复分配然后释放结构体会产生开销，可能会导致巨大的性能瓶颈。解决这个问题的移植方法是为分配的结构体单独维护一个表。当用户不需要某个结构体实例时，将其返回结构体池中。当需要某个结构体实例时，从结构体池中获取一个对象。如果结构体池中没有可用的结构体，就会自动动态分配一个结构体实例。

## 二、结构体宏
### 1、offsetof宏
   
offsetof根据结构体成员的类型和成员名来计算该成员距结构体首地址的偏移量。宏定义如下：


```
#define offsetof(type, member) ((size_t)(&((type *)0)->member))
```

    ( (TYPE *)0 )： 0地址强制 "转换" 为 TYPE结构类型的指针

    ((TYPE *)0)->MEMBER ：访问TYPE结构中的MEMBER数据成员

    &( ( (TYPE *)0 )->MEMBER)：取出TYPE结构中的数据成员MEMBER的地址

    (size_t)(&(((TYPE*)0)->MEMBER))：结果转换为size_t类型

offsetof宏首先将0转换为结构体指针类型，然后引用成员变量并取其地址。由于结构体首地址为0，所以成员变量的地址即为成员距结构体首地址的偏移量。

现代编译器中常用


```
#define offsetof(type, member) __builtin_offsetof(type, member)
```


### 2、container_of宏
    
container_of根据成员地址、结构体类型和成员名来计算结构体的首地址，其定义如下：


```
#define container_of(ptr, type, member) ({ \
const typeof( ((type *)0)->member ) *__mptr = (ptr); \
(type *)( (char *)__mptr - offsetof(type,member) );})
```


创建一个结构体，用offsetof和container_of获取结构体的地址和内部成员相对于结构体地址的偏移量。

代码实例：


```
#include <stdio.h>

#define offsetof(type, member) ((size_t)&((type *)0)->member)

#define container_of(ptr, type, member) ({ \

    const typeof( ((type *)0)->member ) *__mptr = (ptr); \

    (type *)( (char *)__mptr - offsetof(type,member) );})

 
struct test{
    char a;
    int b;
    short c;
    char *p;
    double d;
}__attribute__((aligned(4)));

 

int main(int argc, char **argv)
{

    struct test s;

    printf("offset a=%lu\n",offsetof(struct test,a));
    printf("offset b=%lu\n",offsetof(struct test,b));
    printf("offset c=%lu\n",offsetof(struct test,c));
    printf("offset p=%lu\n",offsetof(struct test,p));
    printf("offset d=%lu\n",offsetof(struct test,d));

    printf("s=%p\n",container_of(&s.a,struct test,a));
    printf("s=%p\n",container_of(&s.p,struct test,p));
    return 0;
}
```


运行结果：


```
offset a=0
offset b=4
offset c=8
offset p=16
offset d=24
s=0x7fffc8590cf0
s=0x7fffc8590cf0
```




