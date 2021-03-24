---
description: >-
  libmill是一个基于c语言开发的go风格协程库，实现了go风格的协程操作、chan操作、非阻塞的网络io操作等等，是一个不错的linux平台下c风格协程库实现。如果你想从0到1的快速了解如何开发一个协程库，libmill将是一个非常不错的案例；如果你想更深入地了解go，libmill里面也借鉴了go的一些设计思想；或者你想在生产环境中使用，开发者也提供了一个更健壮的版本libdill。
---

# 单向链表: slist

**mill\_slist**，它实现的是一个单链表，它也实现了**pop**、**push**、**push\_back**操作，这意味着也可以把它当做stack、queue来使用。

如何当做stack来使用？push、pop！ 如何当做queue来使用？push\_back、pop！

对于当做单链表来使用，其使用方式与list类似，前面也分析了mill\_list的相关实现，这里mill\_slist的实现思路与也之类似。

**slist.h**

```c
struct mill_slist_item {
    struct mill_slist_item *next;
};

struct mill_slist {
    struct mill_slist_item *first;
    struct mill_slist_item *last;
};

/* Initialise the list. To statically initialise the list use = {0}. */
void mill_slist_init(struct mill_slist *self);

/* True is the list has no items. */
#define mill_slist_empty(self) (!((self)->first))

/* Returns iterator to the first item in the list or NULL if the list is empty. */
#define mill_slist_begin(self) ((self)->first)

/* Returns iterator to one past the item pointed to by 'it'. If there are no more items returns NULL. */
#define mill_slist_next(it) ((it)->next)

/* Push the item to the beginning of the list. */
void mill_slist_push(struct mill_slist *self, struct mill_slist_item *item);

/* Push the item to the end of the list. */
void mill_slist_push_back(struct mill_slist *self, struct mill_slist_item *item);

/* Pop an item from the beginning of the list. */
struct mill_slist_item *mill_slist_pop(struct mill_slist *self);
```

**slist.c**

```c
// slist初始化
void mill_slist_init(struct mill_slist *self) {
    self->first = NULL;
    self->last = NULL;
}

// slist push操作（插到链表头部）
void mill_slist_push(struct mill_slist *self, struct mill_slist_item *item) {
    item->next = self->first;
    self->first = item;
    if(!self->last)
        self->last = item;
}

// slist push_back（插到链表尾部）
void mill_slist_push_back(struct mill_slist *self,
      struct mill_slist_item *item) {
    item->next = NULL;
    if(!self->last)
        self->first = item;
    else
        self->last->next = item;
    self->last = item;
}

// slist pop（从链表头部pop）
struct mill_slist_item *mill_slist_pop(struct mill_slist *self) {
    if(!self->first)
        return NULL;
    struct mill_slist_item *it = self->first;
    self->first = self->first->next;
    if(!self->first)
        self->last = NULL;
    return it;
}
```

