---
layout: post
title: Redis_List链表的实现
categories: redis
---

读书笔记02[《Redis设计与实现》](http://item.jd.com/11486101.html)

-------

链表是`Redis`中常用的数据结构，链表是实现列表类型的基础数据结构，在Redis中至关重要。

除实现列表类型外，链表还被用在很多内部模块，比如服务器模块保存客户端、保存命令，事件模块中保存事件等场景。

Redis内部采用一种双端链表的实现。

### 链表实现

**节点**

每个节点用一个`adlist.h/listNode`结构来表示：

```
 // 双端链表节点
typedef struct listNode {
    // 前置节点
    struct listNode *prev;
    // 后置节点
    struct listNode *next;
    // 节点值
    void *value;
} listNode;
```

**链表**

链表结构list提供表头指针、表尾指针、长度计数器，以及通过`dup`,`free`,`match`等函数指针实现的多态链表操作函数,使其可以保存多种不同类型的节点值。

```
// 双端链表结构
typedef struct list {
    // 表头节点
    listNode *head;
    // 表尾节点
    listNode *tail;
    // 节点值复制函数
    void *(*dup)(void *ptr);
    // 节点值释放函数
    void (*free)(void *ptr);
    // 节点值对比函数
    int (*match)(void *ptr, void *key);
    // 链表所包含的节点数量
    unsigned long len;
} list;
```

**链表迭代器**

Redis也提供一个简单链表迭代器，内部包含一个节点指针和迭代方向，迭代方向分为两个，从头到尾，从尾到头。

```
typedef struct listIter {
    // 当前迭代到的节点
    listNode *next;
    // 迭代的方向
    int direction;
} listIter;
```

其中direction代表迭代器的方向：

*  `adlist.h/AL_START_HEAD` ，从表头到表尾的迭代；
*  `adlist.h/AL_START_TAIL` ，从表尾到表头的迭代；

Redis中提供若干个关于迭代器操作的API,比如:

* listGetIterator
* listReleaseIterator
* listRewind
* listRewindTail
* listNext

### 链表操作API

首先，Redis通过宏定义实现了一部分链表API，如下：

```
// 返回给定链表所包含的节点数量，时间复杂度O(1)
#define listLength(l) ((l)->len)
// 返回给定链表的表头节点,时间复杂度O(1)
#define listFirst(l) ((l)->head)
// 返回给定链表的表尾节点，时间复杂度O(1)
#define listLast(l) ((l)->tail)
// 返回给定节点的前置节点，时间复杂度O(1)
#define listPrevNode(n) ((n)->prev)
// 返回给定节点的后置节点，时间复杂度O(1)
#define listNextNode(n) ((n)->next)
// 返回给定节点的值，时间复杂度O(1)
#define listNodeValue(n) ((n)->value)
// 将链表 l 的值复制函数设置为 m，时间复杂度O(1)
#define listSetDupMethod(l,m) ((l)->dup = (m))
// 将链表 l 的值释放函数设置为 m，时间复杂度O(1)
#define listSetFreeMethod(l,m) ((l)->free = (m))
// 将链表的对比函数设置为 m，时间复杂度O(1)
#define listSetMatchMethod(l,m) ((l)->match = (m))
// 返回给定链表的值复制函数，时间复杂度O(1)
#define listGetDupMethod(l) ((l)->dup)
// 返回给定链表的值释放函数，时间复杂度O(1)
#define listGetFree(l) ((l)->free)
// 返回给定链表的值对比函数，时间复杂度O(1)
#define listGetMatchMethod(l) ((l)->match)
```

然后，在adlist.h中定义了其他一些api，比如：

|函数   |作用   |时间复杂度   |
| ------------ | ------------ | ------------ |
| `list *listCreate(void);`  | 创建一个空链表  | O(1)  |
| `void listRelease(list *list);`  |释放链表及所有节点   | O(N)  |
| `list *listAddNodeHead(list *list, void *value);`  | 添加一个定值value的节点到链表头部  |O(1)   |
| `list *listAddNodeTail(list *list, void *value);`  | 添加一个定值value的节点到链表尾部  |O(1)   |
| `list *listInsertNode(list *list, listNode *old_node, void *value, int after);`  |创建一个包含值 value 的新节点，并将它插入到 old_node 的之前或之后   | O(1)  |
| `void listDelNode(list *list, listNode *node);`  |从链表 list 中删除给定节点 node    | o(1)  |
| `listIter *listGetIterator(list *list, int direction);`  |为给定链表创建一个迭代器   | O(1)  |
| `listNode *listNext(listIter *iter);`  | 返回迭代器当前所指向的节点  | O(1)  |
| `void listReleaseIterator(listIter *iter);`  |释放迭代器   |  O(1) |
| `list *listDup(list *orig);`  | 复制整个链表  | O(N)  |
| `listNode *listSearchKey(list *list, void *key);`  | 查找链表 list 中值和 key 匹配的节点  | O(N) |
| `listNode *listIndex(list *list, long index);`  |返回链表在给定索引上的值   | O(N)  |
| `void listRewind(list *list, listIter *li);`  |创建一个指针头部的迭代器   | O(1)  |
| `void listRewindTail(list *list, listIter *li);`  | 创建一个指向尾部的迭代器  | O(1)  |
| `void listRotate(list *list);`  |将尾部节点挪动到头部上   | O(1)  |

最后整个`adlist.c`的注释可见：黄健宏老师的工作，[huangz1990/redis-3.0-annotated](https://github.com/huangz1990/redis-3.0-annotated/blob/unstable/src/adlist.c)

### 小结

Redis链表特性：

* 高效访问头部尾部，时间复杂度O(1)，是高效实现`LPUSH,RPOP`等命令的关键。
* 属性`len`让计算链表长度的时间复杂度为O(1)，所以命令`LEN`也很高效。
* 可以前后两个方向遍历


### Reference

1. 黄健宏. Redis设计与实现[M]. 机械工业出版社, 2014.


