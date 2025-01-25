---
title: 【Redis】数据结构和对象
date: 2023-03-29 19:21:05
categories: Redis
tags: Redis
---

### 开篇

本文介绍`redis`中的数据结构和对象

<!-- more --> 

### 简单动态字符串

`Redis`没有使用C语言传统的字符串表示，而是自己构建了一种名为简单动态字符串的抽象类型，用在可以被修改的字符串值，比如包含字符串值的键值对。

#### SDS的定义

每个 sds.h/sdshdr结构表示一个 SDS值:

```c
struct sdshdr {
    // 记录已使用字节的数量
    int len;
    // 记录buf数组中未使用字节的数量
    int free;
    // 字节数组，用于保存字符串
    chat buf[];
}
```

#### SDS 与 C 字符串的区别

根据传统，C语言使用长度为N+1的字符数组来表示长度为N的字符串，并且字符数组的最后一个元素总是空字符·\0’。C语言使用的这种简单的字符串表示方式，并不能满足Rdis对字符串在安全性、效率以及功能方面的要求。

- 常数复杂度获取字符串长度
- 杜绝缓冲区溢出
- 减少修改字符串时带来的内存重分配次数
- 二进制安全
- 兼容部分C字符串函数

### 链表

链表提供了高效的节点重排能力，以及顺序性的节点访问方式，并且可以通过增删节点来灵活地调整链表的长度。作为一种常用数据结构，链表内置在很多高级的编程语言里面，因为Rdis使用的C语言并没有内置这种数据结构，所以Redis构建了自己的链表实现。

使用场景：

- 列表键
- 发布订阅
- 慢查询
- 监视器
- 服务端保存各个客户端的状态信息
- 使用链表来构建客户端输出缓冲区

#### 链表和链表节点的实现

每个链表节点使用一个 adlist.h/listNode结构来表示

```c
typedef struct listNode {
    // 前置节点
    struct listNode *prev;
    // 后置节点
    struct listNode *next;
    // 节点的值
    void *value;
}
```

虽然仅仅使用多个listNode结构就可以组成链表，但使用adlist.h/list来持有链表的话，操作起来会更方便：

```c
typedef struct list {
    // 表头节点
    listNode *head;
    
    // 表尾结点
    listNode *tail;
    
    // 链表所包含的节点数量
    unsigned long len;
    
    // 节点值复制函数
    void *(*dup) (void *ptr);
    
    // 节点值释放函数
    void *(*free) (void *ptr);
    
    // 节点值对比函数
    int (*match) (void *ptr, void *key);
}
```

Redis的链表实现的特性可以总结如下：

- 双端：链表节点带有prev和next指针，获取某个节点的前置节点和后置节点的复杂度都是O(1)
- 无环：表头节点的prev指针和表尾节点的next指针都指向NULL,对链表的访问以NULL为终点。
- 带表头指针和表尾指针：通过list结构的head指针和tail指针，程序获取链表的表头节点和表尾节点的复杂度为O(1)。
- 带链表长度计数器：程序使用1ist结构的1en属性来对1ist持有的链表节点进行计数，程序获取链表中节点数量的复杂度为O(1)。
- 多态：链表节点使用void*指针来保存节点值，并且可以通过ist结构的dup、free、match三个属性为节点值设置类型特定函数，所以链表可以用于保存各种不
  同类型的值。

### 字典

字典，又称为符号表（symbol table）关联数组(associative array)或映射（map),是一种用于保存键值对(key-value pair)的抽象数据结构。