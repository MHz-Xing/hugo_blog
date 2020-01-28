---
title : "嵌入式Linux-C语言（十二）--单链表"
date : "2019-12-24T10:17:54+08:00"
tags : ["指针","链表","C"]
series : ["嵌入式Linux-C语言"]
categories : ["编程技术"]
draft : false
toc : true
---

## 一、单链表简介
### 1、单链表的结构
    
单链表是一种链式存取的数据结构，用一组地址任意的存储单元存放线性表中的数据元素。

链表中的数据是以节点来表示的，每个节点由两部分构成：一个是数据域，存储数据值，另一个是指针域，存储指向下一个节点的指针。
<!--more-->
![12-1](/images/blog/C/12-1.png)

### 2、单链表的节点

单链表节点的数据结构如下：

```
typedef struct data
{
    unsigned int id;//学生身份ID
    char name[LENGTH];//学生姓名
    char subject[LENGTH];//科目
    unsigned int score;//分数
}DATA,*PDATA;


typedef struct node
{
    unsigned int num;//节点序号，头节点则保存链表的节点数量
    DATA *data;//数据域
    struct node *next;//指针域
}SNode,*PNode;
```

### 3、头指针与头结点的区别

头指针与头结点的区别：

![12-2](/images/blog/C/12-2.png)

## 二、单链表的操作
### 1、单链表的创建

头结点的创建：

**//初始化单链表，创建头结点**

```
SList slist_init(void)
{
    SList head = (Node *)malloc(sizeof(Node));
    if(NULL == head)
    {
        fprintf(stderr, "slist_init function malloc failure.\n");
        return NULL;
    }
    head->next = NULL;
    return head;
}
```

**采用尾部插入节点的方式创建单链表：**

![12-3](/images/blog/C/12-3.png)

**//根据创建的头结点创建有num个节点的单链表（头结点不算）**

```
int slist_create(SList head, unsigned int num)
{
    SList p,r;
    r = head;
    int i;
    for(i = 0; i < num; i++)
    {
        p = (Node *)malloc(sizeof(Node));//创建新节点
        if(NULL == p)
        {
            fprintf(stderr, "slist_create function p malloc failed.\n");
            return ERROR;
        }
        
        p->num = i+1;//将各个节点的序号写入节点成员num
        p->data.id = 100;
        strcpy(p->data.name, "scorpio");
        strcpy(p->data.subject, "English");
        p->data.score = 100;
        r->next = p;
        r = p;
    }
    r->next = NULL;
    head->num = slist_size(head);//将创建链表的大小写入头结点num成员
    return OK;
}
```

### 2、单链表的销毁
**//销毁单链表**

```
void slist_destroy(SList list)
{
    SList p;
    while(list)
    {
        p = list->next;
        free(list);
        list = p;
    }
}
```

### 3、单链表的节点插入操作
![12-4](/images/blog/C/12-4.png)

```
void slist_insert(SList list, unsigned int i,PDATA data)
{
    if(slist_size(list) < i)
    {
        fprintf(stdout, "slist_insert function failed.The list size is too samll.There is no element %u.\n", i);
        return ;
    }

    //创建一个节点
    SList p = (Node *)malloc(sizeof(Node));
    
    //利用传入的data结构体的数据对节点的数据域进行初始化
    p->data.id = data->id;
    strcpy(p->data.name, data->name);
    strcpy(p->data.subject, data->subject);
    p->data.score = data->score;
    
    **//找到节点i**
    SList r = list->next;
    if(NULL == r->next)//链表为空时，直接插入
    {
        //插入节点到链表
        p->next = r->next;
        r->next = p;
    }
    else//链表不为空时，先查找插入的位置
    {
        while(i != r->num)
        {
            r = r->next;
        }
        //插入节点到链表
        p->next = r->next;
        r->next = p;
    }
    
    //顺延链表节点i后的节点的序号
    int n = i + 1;
    r = r->next;
    while(r->next)
    {
        r->num = n++;
        r = r->next;
    }
    r->num = n;//尾节点的序号
    list->num = slist_size(list);//将创建链表的大小写入头结点num成员
}
```

### 4、单链表的节点删除操作
![12-5](/images/blog/C/12-5.png)


```
void slist_delete(SList list, unsigned int i)
{
    if(slist_size(list) < i)
    {
        fprintf(stdout, "slist_insert function failed.The list size is too samll.There is no element %u.\n", i);
        return ;
    }

    SList r = list->next;
    if(NULL == r->next)//链表为空时，不能删除
    {
        fprintf(stderr, "The list is empty.There is no element %u.\n", i);
        return ;
    }
    else//链表不为空时，先查找插入的位置
    {
        while(i != r->num)
        {
        r = r->next;
        }

        //从链表中删除节点
        r->next = r->next->next;
    }

    //顺延链表节点i后的节点的序号

    int n = i;

    while(r->next)
    {
        r->num = n++;
        r = r->next;
    }
    r->num = n;//尾节点的序号
    list->num = slist_size(list);//将创建链表的大小写入头结点num成员
}
```


### 5、获取单链表的长度

**//计算单链表的长度，头节点不计算在长度之内**

```
unsigned int  slist_size(SList list)
{
    unsigned int count;
    SList p = list->next;
    while(p)
    {
        p = p->next;
        count++;
    }
    return count;
}
```

### 6、判断单链表是否为空
**//判断单链表是否为空**

```
int slist_is_empty(SList list)
{
    if(list)
    {
        fprintf(stdout, "list is no empty.\n");
        return 0;
    }
    else
    {
        fprintf(stdout, "list is empty.\n");
        return 1;
    }
}
```

### 7、遍历单链表
**//遍历单链表**

```
void slist_traverse(SList list)
{
    fprintf(stdout, "list size is %u\n", list->num);
    SList p = list->next;//跳过头结点
    while(p)
    {
        fprintf(stdout, "id = %u, name = %s, subject = %s, score = %u\n", p->data.id, p->data.name, p->data.subject, p->data.score);
        p = p->next;
    }
}
```

### 8、单链表的翻转（倒序、逆序）

（1）定义当前节点 current，初始值为第一个节点，current =list->next;

（2）定义当前结点的后继节点pnext， pnext = current->next; 

（3）将当前第一个节点与后继节点断开，作为尾节点，current->next = NULL；

（4）只要 pnext 存在，表明至少有两个节点，进行逆序，执行以下循环：

- A、定义新节点prev，prev是 pnext的后继节点，prev = pnext->next;

- B、把pnext的后继指向current, pnext->next = current;

- C、此时，pnext 实际上已经到了current 前一位成为新的current，所以这个时候 current 结点实际上成为新的 pnext，current = pnext;

- D、此时，新的 current 就是 pnext，current = pnext;

- E、而新的 pnext 就是 prev，pnext = prev;

（5）最后将头结点与current重新连上即可，list->next = current;

```
void slist_reverse(SList list)
{
    if(slist_is_empty(list))
    {
        fprintf(stdout, "The list is empty.No reverse.\n");
        return ;
    }
    
    SList prev, current, pnext;
    current = list->next;//跳过头节点，当前节点为第一个节点
    pnext = current->next;//当前节点的下一个节点
    current->next = NULL;//当前节点断开后续连接，作为尾节点
    while(pnext)
    {
        prev = pnext->next;//将当前节点的下一个节点作为当前节点的前一个节点
        pnext->next = current;//当前节点的前一个节点的指针域指向当前节点
        current = pnext;//将下一个节点作为当前节点
        pnext = prev;//前一个节点作为下一个节点
    }
    list->next = current;//头结点指针域指向当前节点，完成逆序
    
    //重新给逆序后的各节点赋值新的序号
    int i = 1;
    while(current)
    {
        current->num = i++;
        current = current->next;
    }
}
```

参考博文：

[小猪的数据结构辅助教程——2.2 线性表中的单链表 （CSDN coder-pig）](http://blog.csdn.net/coder_pig/article/details/50234425)