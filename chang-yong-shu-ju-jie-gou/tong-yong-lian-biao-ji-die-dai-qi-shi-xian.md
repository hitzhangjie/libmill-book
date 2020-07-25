# 通用链表及迭代器实现

**通用链表及迭代器实现**

**offsetof**可以计算结构体中的成员的offset，如果我们知道一个struct的类型、其成员名、成员地址，我们就可以计算出struct的地址：

```c
#define mill_cont(ptr, type, member) \
        (ptr ? ((type*) (((char*) ptr) - offsetof(type, member))) : NULL)
```

基于此可以进一步实现一个通用链表，怎么搞呢？

```c
struct list_item {
    struct list_item * next;
};
​
struct my_struct {
    void * data; 
    struct list_item * iter;
};
```

我们通过list\_item来构建链表，并在自定义my\_struct中增加一个list\_item成员，将其用作迭代器。当我们希望构建一个my\_struct类型的链表时实际上构建的是list\_item的列表，当我们遍历my\_struct类型的链表时遍历的也是list\_item构成的链表。假如现在遍历到了链表中的某个list\_item item，就可以结合前面提到的mill\_cont\(&item, struct list\_item, iter\)来获得包括该成员的结构体地址，进而就可以访问结构体中的数据成员data。

其实这里Martin Sustrik的实现方式与Linux下的通用链表相关的宏实现类似，只是使用起来感觉更加自然一些，也更容易被接受。

