---
description: >-
  libmill是一个基于c语言开发的go风格协程库，实现了go风格的协程操作、chan操作、非阻塞的网络io操作等等，是一个不错的linux平台下c风格协程库实现。如果你想从0到1的快速了解如何开发一个协程库，libmill将是一个非常不错的案例；如果你想更深入地了解go，libmill里面也借鉴了go的一些设计思想；或者你想在生产环境中使用，开发者也提供了一个更健壮的版本libdill。
---

# 双向链表: list

**1 list.h**

mill\_list是一个双向链表，链表内部的链接通过mill\_list\_item来维护，mill\_list\_item与mill\_cont相当于实现了list中的iterator。

```c
struct mill_list_item {
    struct mill_list_item *next;
    struct mill_list_item *prev;
};

struct mill_list {
    struct mill_list_item *first;
    struct mill_list_item *last;
};

/* Initialise the list. To statically initialise the list use = {0}. */
void mill_list_init(struct mill_list *self);

/* True is the list has no items. */
#define mill_list_empty(self) (!((self)->first))

/* Returns iterator to the first item in the list or NULL if
   the list is empty. */
#define mill_list_begin(self) ((self)->first)

/* Returns iterator to one past the item pointed to by 'it'. */
#define mill_list_next(it) ((it)->next)

/* Adds the item to the list before the item pointed to by 'it'.
   If 'it' is NULL the item is inserted to the end of the list. */
void mill_list_insert(struct mill_list *self, struct mill_list_item *item, struct mill_list_item *it);

/* Removes the item from the list and returns pointer to the next item in the
   list. Item must be part of the list. */
struct mill_list_item *mill_list_erase(struct mill_list *self, struct mill_list_item *item);
```

当我们创建一个list时，我们创建一个struct mill\_list变量，当我们要插入元素到list中时，实际上插入的是一个mill\_list\_item，当我们要遍历一个list时实际上是通过mill\_list.first/last以及mill\_list\_item.next/prev来进行遍历。

不禁要问，我们要保存的链表元素肯定不只是mill\_list\_item啊？struct元素是不是还要包括其他成员？是。这里就要提到**mill\_cont**这个函数了，该方法可以获取包含某个member的struct结构体的地址，如果在自定义struct中额外增加一个成员mill\_list\_item，保存链接关系的时候使用mill\_list\_item成员，访问自定义struct完整信息的时候再通过该成员以及mill\_cont来获取自定义结构体地址，进而解引用访问，这样问题就解决了？

我们要存储的结构体以struct Student为例，来演示一下上述操作：

```c
// 自定义struct
struct Student {
    char *name;
    int age;
    int sex;
    struct list_mill_item item;
};

// 创建链表
struct mill_list students = {0};

// 添加元素到链表
struct Student stu_x = {.name="x", .age=10, .sex=0};
struct Student stu_y = {.name="y", .age=11, .sex=1};

mill_list_init(&students, &stu_x->item, NULL);
mill_list_init(&students, &stu_y->item, NULL);

// 遍历链表元素
struct mill_list_item *iter = students.first;
while(iter) {
    struct Student *stu = (struct Student *)mill_cont(iter, struct Student, item);
    printf("student name:%s, age:%d, sex:%d\n", stu->name, stu->age, stu->sex);

    iter = iter->next;
}
```

**2 list.c**

```c
//初始化一个空链表
void mill_list_init(struct mill_list *self)
{
    self->first = NULL;
    self->last = NULL;
}

// 在链表self中在元素it前面插入item
void mill_list_insert(struct mill_list *self, struct mill_list_item *item, struct mill_list_item *it)
{
    item->prev = it ? it->prev : self->last;
    item->next = it;
    if(item->prev)
        item->prev->next = item;
    if(item->next)
        item->next->prev = item;
    if(!self->first || self->first == it)
        self->first = item;
    if(!it)
        self->last = item;
}

// 从链表self中删除元素item
struct mill_list_item *mill_list_erase(struct mill_list *self, struct mill_list_item *item)
{
    struct mill_list_item *next;

    if(item->prev)
        item->prev->next = item->next;
    else
        self->first = item->next;
    if(item->next)
        item->next->prev = item->prev;
    else
        self->last = item->prev;

    next = item->next;

    item->prev = NULL;
    item->next = NULL;

    return next;
}
```

