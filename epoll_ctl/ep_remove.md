```c
//删除一个epitem节点
static int ep_remove(struct eventpoll *ep, struct epitem *epi)  
{  
    unsigned long flags;  
    struct file *file = epi->ffd.file;  

    /* 
     * Removes poll wait queue hooks. We _have_ to do this without holding 
     * the "ep->lock" otherwise a deadlock might occur. This because of the 
     * sequence of the lock acquisition. Here we do "ep->lock" then the wait 
     * queue head lock when unregistering the wait queue. The wakeup callback 
     * will run by holding the wait queue head lock and will call our callback 
     * that will try to get "ep->lock". 
     */  
    //清理在资源等待队列上的等待项：
    //该函数负责从被监控文件的等待队列中移除 epoll 设置的等待项
    //这些等待项是在 ep_insert 时通过 ep_ptable_queue_proc 添加的
    ep_unregister_pollwait(ep, epi);  

    /* Remove the current item from the list of epoll hooks */  
    spin_lock(&file->f_lock);  
    if (ep_is_linked(&epi->fllink))  
        list_del_init(&epi->fllink);  
    spin_unlock(&file->f_lock);  

    rb_erase(&epi->rbn, &ep->rbr); //从红黑树移除对这个节点的管理  

    spin_lock_irqsave(&ep->lock, flags);  
    if (ep_is_linked(&epi->rdllink))  
        list_del_init(&epi->rdllink);  
    spin_unlock_irqrestore(&ep->lock, flags);  

    /* At this point it is safe to free the eventpoll item */  
    kmem_cache_free(epi_cache, epi);   //删除节点对象

    atomic_long_dec(&ep->user->epoll_watches);  

    return 0;  
}  
```