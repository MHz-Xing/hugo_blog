---
title : "嵌入式Linux-C语言（三）--指针与函数"
date : "2019-12-24T09:38:52+08:00"
tags : ["指针","函数","C"]
series : ["嵌入式Linux-C语言"]
categories : ["编程技术"]
draft : false
toc : true
---
  
  指针对函数的功能有巨大的贡献，指针能够将数据传递给函数，并且允许函数对数据进行修改。指针对于函数的作用主要有两方面：将指针传递给函数和声明函数指针。

<!--more-->
## 一、程序的栈和堆

程序的栈和堆是C语言程序运行的运行时元素。

### 1、程序栈
程序栈是支持函数执行的内存区域，通常和堆共享一块内存区域，通常程序栈占据内存区域的高地址区域，堆用内存区域的低地址区域。程序栈存放栈帧，栈帧存放函数参数和局部变量。调用函数时，函数的栈帧被push到栈上，栈生长长出一个栈帧，当函数终止时，函数的栈帧从程序栈上pop出栈，栈帧所使用的内存不会被清理，但可能会被推倒程序栈上的另一个函数的栈帧所覆盖。动态分配的内存来自于堆，堆向高地址区域生长，随着内存的分配和释放，堆中会布满碎片。

另外，需要特别指出的是，这里的地址高低实际上我没有找到十分明确的规范，大体上来说可以这样归纳：
    
X86体系，按照栈帧高地址向低地址方向增长，且位于高地址方向，堆则相反。
ARM体系，没有十分明确的规定，需要按照不同OS规范来确认，大部分情况下，和X86保持一致，特别的对于裸机系统，像STM32也同样按照栈帧高地址向低地址方向增长，且位于高地址方向，堆则相反。
[知乎相关讨论](https://www.zhihu.com/question/36103513?sort=created)

### 2、栈帧
  
栈帧的组成：

**A、返回地址**

函数完成后要返回的程序内部地址

**B、局部数据存储**

为局部变量分配的内存

**C、参数存储**

为函数参数分配的内存

**D、栈指针和基指针**

运行时系统用来管理栈的指针

栈指针通常指向栈的顶部，基指针（帧指针）通常存在并指向栈帧内部的地址

ARM的栈帧布局如下：

![3-1](/images/blog/C/3-1.png)


## 二、通过指针传递和返回数据

### 1、用指针来传递数据

函数中用指针变量作为参数来传递数据可以在函数中修改数据，如果不需要修改数据，则将指针变量限制为const类型。典型的函数应用如下：

```
char *strcpy(char *dest, const char *src);
```

如果函数用值传递数据，则在函数中将无法修改数据。

### 2、返回指针

返回指针需要返回的类型是某种数据类型的指针。

从函数返回指针存在的问题：

- 返回未初始化的指针

- 返回指向无效地址的指针。

- 返回局部变量的指针

- 返回指针但是没有释放内存

从函数返回动态分配的内存，在使用完内存后必须释放，否则会造成内存泄漏。

函数返回局部数据的指针或局部变量是错误的，函数返回后，局部数据所在的栈帧将会被弹出程序栈，保存在栈帧上的数据极易被后续调用函数的栈帧覆盖。通过将局部变量声明为static类型，可以将局部数据、变量的作用域限定在函数内部，但分配在栈外（data数据段），可以避免局部数据、变量被其他函数的栈帧覆盖。

### 3、传递指针的指针

将指针传递给函数时，传递的是指针变量的值，如果需要修改原指针而不是指针变量的副本，就需要传递指针的指针。

### 4、函数参数的求值顺序

函数参数的求值顺序依赖于编译器的实现。

GCC编译器

```
#include <stdio.h>

void fun(int i, int k)
{
    printf("%d %d\n", i, k);
}

int main(int argc, char *argv[])
{
    int k = 1;
    int i = 1;
    int j = 1;
    int l = 1;
    j = j++ + j++;//j = 4
    l = ++l + ++l;//l = 6
    fun(++k,++k);//3,3
    fun(i++,i++);//2,1
    return 0;
}
```

函数参数的入栈顺序

函数调用发生时，参数会被传递给被调用的函数，返回值则返回给函数调用者。

函数调用约定描述了函数参数如何传递到栈以及栈的维护方式。

调用约定用于库调用和库开发的时候。

从右到左依次入栈：__stdcall, __cdecl, __thiscall

从左到右依次入栈：__pascal, __fastcall

当C语言调用其他语言如pascal语言的库函数时需要显示声明调用约定。

### 5、可变参数的函数
    
C语言中可以定义参数可变的函数，参数可变函数的实现依赖于stdarg.h头文件。

- va_list:参数集合

- va_arg:取具体参数值

- va_start:标识参数的开始

- va_end:标识参数的结束

- 可变参数必须从头到尾按照顺序逐个访问

- 参数列表中至少要存在一个确定的命名参数

- 可变参数函数无法确定实际存在的参数数量

- 可变参数函数无法确定参数的实际类型，va_arg如果指定了错误的参数类型结构将是不确定的。


```
#include <stdio.h>
#include <stdarg.h>

float average(int n, ...)
{
    va_list args;
    int i = 0;
    float sum = 0;
    va_start(args, n);
    for(i=0; i<n; i++)
    {
        sum += va_arg(args, int);
    }

    va_end(args);
    return sum / n;
}

int main()
{
    printf("%f\n", average(5, 1, 2, 3, 4, 5));
    printf("%f\n", average(4, 1, 2, 3, 4));
    return 0;
}
```


## 三、函数指针

函数指针是持有函数地址的指针变量，指向函数地址的指针变量。

### 1、函数指针的声明

函数指针变量的声明一般格式为;

数据类型 (*指针变量名)();


```
Void * fun(void);//返回void*类型指针的函数

void (*pFun)(void);//函数指针

void *(*pFun)(void *, void *);//函数指针

typedef void(*Fun)(void);//定义一个函数指针类型Fun

Fun pfun = fun;//将函数fun的地址赋值给函数指针pfun

Fun pfun = &fun;//将函数fun的地址赋值给函数指针pfun

void *(*pf[5])(void);//函数指针数组
```


### 2、函数指针的使用

调用函数指针的一般流程：

1. 定义函数指针变量
2. 被调函数的入口地址（函数名）赋予该函数指针变量
3. 用函数指针变量形式调用函数
4. 函数指针变量形式调用函数的一般形式为：(*指针变量名) (实参列表)

函数调用和函数指针变量形式调用函数方式如下：
```
    fun();//函数调用

    (*fun)();//函数指针变量形式调用函数

    (**fun)();//函数指针变量形式调用函数

    pfun();//函数指针形式调用函数

    (*pfun)();//函数指针形式调用函数

    (**pfun)();//函数指针形式调用函数
```

函数指针变量不能进行算术运算，函数指针的偏移是毫无意义的。函数调用中(*指针变量名)的两边的括号不可少，其中的*不应该理解为求值运算，*只是一种表示符号。

### 3、传递函数指针

将函数指针变量作为函数的参数进行传递，使程序代码变得更加灵活。

```
int add(int num1, int num2)
{
    return num1 + num2;
}

int sub(int num1, int num2)
{
    return num1 - num2;
}

typedef int (*pFun)(int, int);

int compute(pFun operation, int num1, int num2)
{
    return operation(num1, num2);
}

compute(add, 5, 6);

compute(sub, 10,2);
```
add、sub函数的地址被传递给compute函数作为参数，compute使用add、sub函数的地址调用对应的操作。

### 4、返回函数指针

返回函数指针需要把函数的返回类型声明为函数指针。

```
int add(int num1, int num2)
{
    return num1 + num2;
}

int sub(int num1, int num2)
{
    return num1 - num2;
}

typedef int (*pFun)(int, int);

pFun select(char opcode)
{
    switch(opcode)
    {
    case ‘+’:
        return add;
    case ‘-’:
        return sub;
    }
}

int evaluate(char opcode, int num1, num2)
{
    pFun operation = select(opcode);
    return operation(num1, num2);
}

evaluate(‘+’, 5, 6);
evaluate(‘-’, 10, 6);
```


通过输入一个字符和两个操作数就可以进行相应的计算

### 5、函数指针数组使用

函数指针数组可以基于某些条件选择要执行的函数，函数指针数组声明如下：

第一种声明：

```
typedef int (*operation)(int, int);
operation operations[128] = {NULL};
```

第二种声明：

```
int (*operations[128]) (int, int) = {NULL};
```

函数指针数组声明后可以将某一类操作函数赋值给数组。

```
operations[‘+’] = add;
operations[‘-’] = sub;

int evaluate_array(char opcode, int num1, num2)
{
    pFun operation = operations[opcode];
    return operation(num1, num2);
}

evaluate_array(‘+’, 6 ,9);
```

### 6、函数指针转换

将指向函数的指针变量转换为其它类型的指针变量。无法保证函数指针和数据指针相互转换后正常工作。

## 四、函数与宏

宏是由预处理器直接展开的，编译器不知道宏的存在，函数是由编译器直接编译的实体，调用行为由编译器决定。多次使用宏会导致可执行程序体积变大，函数是跳转执行的，内存中只有一份函数体存在。宏的效率较高，没有调用开销；函数调用时会记录活动记录，有调用开销。

宏的效率比函数高，但宏是文本替换，参数无法进行类型检查，因此可以使用函数完成的功能绝对不能使用宏，并且宏的定义中不能使用递归定义。

实例代码：

```
#include <stdio.h>
#include <malloc.h>

#define MALLOC(type, x)   (type*)malloc(sizeof(type)*x)
#define FREE(p)           (free(p), p=NULL)

#define LOG_INT(i)        printf("%s = %d\n", #i, i)
#define LOG_CHAR(c)       printf("%s = %c\n", #c, c)
#define LOG_FLOAT(f)      printf("%s = %f\n", #f, f)
#define LOG_POINTER(p)    printf("%s = %p\n", #p, p)
#define LOG_STRING(s)     printf("%s = %s\n", #s, s)

#define FOREACH(i, n)     while(1) { int i = 0, l = n; for(i=0; i < l; i++)

#define BEGIN             {
#define END               } break; }


int main()
{
    int* pi = MALLOC(int, 5);
    char* str = "D.T.Software";

    LOG_STRING(str);
    LOG_POINTER(pi);

    FOREACH(k, 5)
    BEGIN
        pi[k] = k + 1;
    END

    FOREACH(n, 5)

    BEGIN
        int value = pi[n];
        LOG_INT(value);
    END

    FREE(pi);
    
    LOG_POINTER(pi);
    
    return 0;
}
```


## 五、递归函

递归是数学上一种分而自治的思想。递归需要有边界，当边界条件不满足时，递归继续进行；当边界条件满足时，递归终止。

递归函数是函数体内自我调用的函数，是递归数学思想在程序设计上的应用。递归函数必须有递归出口，函数没有递归出口将导致无限递归，造成程序栈溢出而崩溃。

递归模型的一般表示方法：

![3-2](/images/blog/C/3-2.png)


### 1、递归实现求字符串长度函数

```
int strlen_r(const char* s)
{
    if( *s )
    {
        return 1 + strlen_r(s+1);
    }
    else
    {
        return 0;
    }
}
```

### 2、斐波那契数列解法

斐波那契数列表示如下：1，1，2，3，5，8，13，21，...

![3-3](/images/blog/C/3-3.png)


```
int fac(int n)
{
    if( n == 1 )
    {
        return 1;
    }
    else if( n == 2 )
    {
        return 1;
    }
    else
    {
        return fac(n-1) + fac(n-2);
    }
    return -1;
}
```


### 3、递归实现汉诺塔
    
汉诺塔问题：

将木块借助B柱由A柱移到C柱

每次只能移动一个木块

小木块只能放在大木块之上


![3-4](/images/blog/C/3-4.png)

解决方法：

将n-1个木块借助C柱由A柱移动到B柱

将最底层的木块直接移动到C柱

将B柱上的n-1歌木块借助A柱由B柱移动到C柱

```
#include <stdio.h>

void han_move(int n, char a, char b, char c)
{
    if( n == 1 )
    {
        printf("%c --> %c\n", a, c);
    }
    else
    {
        han_move(n-1, a, c, b);
        han_move(1, a, b, c);
        han_move(n-1, b, a, c);
    }
}

int main()
{
    han_move(3, 'A', 'B', 'C');
    return 0;
}
```


 

## 六、函数设计原则

函数设计的一般原则：

A、函数是一个独立功能的模块

B、函数名要在一定程度上反映函数的功能

C、函数参数名要能够体现参数的意义

D、尽量避免在函数中使用全局变量

E、当函数参数不应该在函数内存被修改时，参数应该使用const声明

F、如果参数是指针，且仅用作输入参数，参数应该使用const声明

G、不能省略函数返回值，无返回值应声明位void

H、对参数进行有效性检查，对于指针参数的有效性检查很重要

I、不要返回指向栈内存的指针，栈内存在函数结束时将自动释放

J、函数规模要小，尽量控制在80行内

K、函数避免有过多参数，控制在4个采纳数之内