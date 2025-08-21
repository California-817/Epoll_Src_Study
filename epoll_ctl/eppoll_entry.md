```c
// 与一个文件上的wait_queue_head相关联，因为同一个文件可能有多个等待的事件，
// 这些事件可能使用不同的等待队列
// 钩子函数
struct wait_queue_entry {
    unsigned int        flags;
    void            *private;
    wait_queue_func_t   func;
    struct list_head    entry;
};

//与具体资源进行交互的等待项
struct eppoll_entry {
    // 插入所属epitem节点的队列
    struct list_head llink;
    // 关联的epitem
    struct epitem *base;
    // 插入资源等待队列的节点
    wait_queue_entry_t wait;
    // 指向资源等待队列的头指针所在结构体
    wait_queue_head_t *whead;
};
```