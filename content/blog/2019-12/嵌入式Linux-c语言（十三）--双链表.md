---
title : "嵌入式Linux-C语言（十三）--双链表"
date : "2019-12-24T10:21:20+08:00"
tags : ["指针","链表","C"]
series : ["嵌入式Linux-C语言"]
categories : ["编程技术"]
draft : false
toc : true
---

## 一、双链表简介
### 1、双链表的结构
    
双链表是链表的一种，由节点组成，每个数据结点中都有两个指针，分别指向直接后继和直接前驱。
<!--more-->
![13-1](/images/blog/C/13-1.png)

### 2、双链表的节点

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
    struct node *prev;
    unsigned int num;//节点序号，头节点则保存链表的节点数量
    DATA data;//数据
    struct node *next;
}Node,*DList;
```

![13-2](/images/blog/C/13-2.png)

## 二、双链表的操作

### 1、双链表的创建

**双链表头结点的创建：**

```
DList dlist_init(void)
{
    DList head = (Node *)malloc(sizeof(Node));
    if(NULL == head)
    {
        fprintf(stderr, "dlist_init function malloc failure.\n");
        return NULL;
    }
    head->next = NULL;
    head->prev = NULL;
    return head;
}
```

**采用尾部插入节点的方式创建双链表**
```
int dlist_create(DList head, unsigned int num)
{
    DList p,r;
    r = head;
    int i;
    for(i = 0; i < num; i++)
    {
        p = (Node *)malloc(sizeof(Node));//创建新节点
        if(NULL == p)
        {
            fprintf(stderr, "dlist_create function p malloc failed.\n");
            return ERROR;
        }
        p->num = i+1;//将各个节点的序号写入节点成员num
        p->data.id = 1000 + i;
        strcpy(p->data.name, "scorpio");
        strcpy(p->data.subject, "English");
        p->data.score = 100;
        r->next = p;
        p->prev = r;
        r = p;
    }
    r->next = NULL;
    head->num = dlist_size(head);//将创建链表的大小写入头结点num成员
    return OK;
}
```

### 2、双链表的销毁

```
void dlist_destroy(DList list)
{
    DList p;
    while(list)
    {
        p = list->next;
        free(list);
        list = p;
    }
}
```

### 3、双链表的节点插入操作
![13-3](/images/blog/C/13-3.png)

```
void dlist_insert(DList list, unsigned int i, PDATA data)
{
    if(dlist_size(list) < i)
    {
        fprintf(stdout, "dlist_insert function failed.The list size is too samll.There is no element %u.\n", i);
        return ;
    }
    
    //创建一个节点
    DList p = (Node *)malloc(sizeof(Node));
    
    //利用传入的data结构体的数据对节点的数据域进行初始化
    p->data.id = data->id;
    strcpy(p->data.name, data->name);
    strcpy(p->data.subject, data->subject);
    p->data.score = data->score;
    
    //找到节点i
    DList r = list->next;
    if(NULL == r->next)//链表为空时，直接插入
    {
        //插入节点到链表
        p->next = r->next;
        r->next = p;
        p->prev = r;
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
        p->prev = r;
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
    list->num = dlist_size(list);//将链表的大小写入头结点num成员
}
```

### 4、双链表的节点的删除操作
![13-4](/images/blog/C/13-4.png)

```
void dlist_delete(DList list, unsigned int i)
{
    if(dlist_size(list) < i)
    {
        fprintf(stdout, "dlist_insert function failed.The list size is too samll.There is no element %u.\n", i);
        return ;
    }
    DList r = list->next;
    
    //查找要删除节点的位置
    while(i != r->num)
    {
        r = r->next;
    }
    DList p = r->next;
    
    //从链表中删除节点
    r->next->next->prev = r;
    r->next = r->next->next;
    free(p);
    
    //顺延链表节点i后的节点的序号
    int n = i;
    while(r->next)
    {
        r->num = n++;
        r = r->next;
    }
    r->num = n;//尾节点的序号
    list->num = dlist_size(list);//将创建链表的大小写入头结点num成员
}
```

### 5、获取双链表的长度

```
unsigned int dlist_size(DList list)
{
    unsigned int count = 0;
    DList p = list->next;
    while(p)
    {
        p = p->next;
        count++;
    }
    return count;
}
```
### 6、判断双链表是否为空

```
int dlist_is_empty(DList list)
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
### 7、遍历双链表

```
void dlist_traverse(DList list)
{
    fprintf(stdout, "list size is %u\n", list->num);
    DList p = list->next;//跳过头结点
    while(p)
    {
        fprintf(stdout, "num = %u, id = %u, name = %s, subject = %s, score = %u\n", p->num, p->data.id, p->data.name, p->data.subject, p->data.score);
        p = p->next;
    }
}
```
### 8、双链表的翻转（逆序、倒序）

```
void dlist_reverse(DList list)
{
    if(dlist_is_empty(list))
    {
        fprintf(stdout, "The list is empty.No reverse.\n");
        return ;
    }
    
    DList prev, current, pnext;
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

