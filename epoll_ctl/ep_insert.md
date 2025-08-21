```c
//ep_insert()在epoll_ctl()中被调用, 完成往epollfd里面添加一个监听fd的工作
static int ep_insert(struct eventpoll *ep, struct epoll_event *event,  
                     struct file *tfile, int fd)  
{  
    int error, revents, pwake = 0;  
    unsigned long flags;  
    long user_watches;  
    struct epitem *epi;  
    struct ep_pqueue epq; 

    // ep_pqueue这个结构体详解
    /* 
    typedef struct poll_table_struct {
    // 函数指针
    poll_queue_proc _qproc;
    // unsigned
    __poll_t _key;
    } poll_table;

    struct ep_pqueue { 
        poll_table pt; 
        struct epitem *epi; 
    }; 
    */  

    // 获取这个epoll的监听文件描述符个数
    user_watches = atomic_long_read(&ep->user->epoll_watches);  
    if (unlikely(user_watches >= max_user_watches)) { //超出最大个数限制  
        return -ENOSPC;  
    }  

    // 分配初始化 epi  创建一个epitem
    if (!(epi = kmem_cache_alloc(epi_cache, GFP_KERNEL))) {  
        return -ENOMEM;  
    }  
    //初始化这个新节点
    INIT_LIST_HEAD(&epi->rdllink);  
    INIT_LIST_HEAD(&epi->fllink);  
    INIT_LIST_HEAD(&epi->pwqlist);  
    epi->ep = ep; //所属epoll模型  

    // 初始化红黑树中的key---用于在红黑树中查找该节点
    ep_set_ffd(&epi->ffd, tfile, fd);  

    // 记录订阅事件 来源于内核栈上的结构体 
    epi->event = *event;  
    epi->nwait = 0;  
    epi->next = EP_UNACTIVE_PTR;  

    // 初始化临时的 epq  
    epq.epi = epi;  
    // 设置这个epq的函数指针 钩子函数 epq.pt._qproc=ep_ptable_queue_proc
    init_poll_funcptr(&epq.pt, ep_ptable_queue_proc);  
    // 设置事件掩码  
    epq.pt._key = event->events; 

    // 在这个poll内部会调用前面注册的ep_ptable_queue_proc  
    // 在ep_ptable_queue_proc函数中向资源等待队列注册等待项并注册该等待项回调函数为ep_poll_callback, 并返回当前文件的状态  
    //执行poll钩子函数。epoll是一种机制 支持epoll的其他模块 需要实现专属poll钩子函数

// 通用的poll_wait 函数, 文件的f_ops->poll 通常会调用此函数 
/* static inline void poll_wait(struct file * filp, wait_queue_head_t * wait_address, poll_table *p)  
{  
    if (p && p->_qproc && wait_address) {  
        // 调用_qproc 在wait_address 上添加节点和回调函数  
        // 调用 poll_table_struct 上的函数指针向wait_address添加节点, 并设置节点的func  
        // (如果是select或poll 则是 __pollwait, 如果是 epoll 则是 ep_ptable_queue_proc),  
        p->_qproc(filp, wait_address, p);   //这个函数由上层自己实现钩子函数
    }  
} */
    revents = tfile->f_op->poll(tfile, &epq.pt);   // 判断是否已经有事件触发了

    // 检查错误  
    error = -ENOMEM;  
    if (epi->nwait < 0) { // f_op->poll 过程出错  
        goto error_unregister;  
    }  
    // 添加当前的epitem 到文件的f_ep_links 链表  
    spin_lock(&tfile->f_lock);  
    list_add_tail(&epi->fllink, &tfile->f_ep_links);  
    spin_unlock(&tfile->f_lock);  

    // 插入epi 到rbtree     一旦插入，除非显式删除 如调用 ep_remove 否则会一直保留在红黑树中
    ep_rbtree_insert(ep, epi);  

    /* now check if we have created too many backpaths */  
    error = -EINVAL;  
    if (reverse_path_check()) {  
        goto error_remove_epi;  
    }  

    spin_lock_irqsave(&ep->lock, flags);  

    /* 一进来新增节点判断事件revents 发现文件事件已经就绪  直接插入到就绪链表rdllist */  
    if ((revents & event->events) && !ep_is_linked(&epi->rdllink)) {  

        list_add_tail(&epi->rdllink, &ep->rdllist);   //插入rdlist中

         //等待队列非空则唤醒阻塞在该epoll的进程或线程
        if (waitqueue_active(&ep->wq))  
            // 通知sys_epoll_wait , 调用回调函数唤醒sys_epoll_wait 进程  
        {  
            wake_up_locked(&ep->wq);  //唤醒一个线程或进程 
        }  
        // 先不通知调用eventpoll_poll 的进程 :   该epoll被另一个监听 需要唤醒主epoll
        if (waitqueue_active(&ep->poll_wait)) {  
            pwake++;  
        }  
    }  

    spin_unlock_irqrestore(&ep->lock, flags);

    //监听的fd数量++
    atomic_long_inc(&ep->user->epoll_watches); 

    //该epoll被另一个监听 需要唤醒主epoll
    if (pwake)  
        // 安全通知调用eventpoll_poll 的进程  
    {  
        ep_poll_safewake(&ep->poll_wait);  
    }  

    return 0;



    //错误处理的一些逻辑
    error_remove_epi:  
    spin_lock(&tfile->f_lock);  
    // 删除文件上的 epi  
    if (ep_is_linked(&epi->fllink)) {  
        list_del_init(&epi->fllink);  
    }  
    spin_unlock(&tfile->f_lock);  

    // 从红黑树中删除  
    rb_erase(&epi->rbn, &ep->rbr);  

    error_unregister:  
    // 从文件的wait_queue 中删除, 释放epitem 关联的所有eppoll_entry  
    ep_unregister_pollwait(ep, epi);  

    spin_lock_irqsave(&ep->lock, flags);  
    if (ep_is_linked(&epi->rdllink)) {  
        list_del_init(&epi->rdllink);  
    }  
    spin_unlock_irqrestore(&ep->lock, flags);  

    // 释放epi  
    kmem_cache_free(epi_cache, epi);  

    return error;  
}
```