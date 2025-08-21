```c
/* 
 * 该函数在调用f_op->poll()时会被调用. 是epoll实现的一个钩子函数
 * 也就是epoll主动poll某个fd时, 用来将epitem与指定的socketfd等待队列中的等待项关联起来的.
 * 关联的办法就是使用等待队列(waitqueue)
 */
static void ep_ptable_queue_proc(struct file *file, wait_queue_head_t *whead,  //这个whead是具体资源的等待队列 如socket的等待队列
                 poll_table *pt)
{
    //获取到这个epitem结构
    struct epitem *epi = ep_item_from_epqueue(pt);

    // 分配一个eppoll_entry  等待项
    struct eppoll_entry *pwq;
    if (epi->nwait >= 0 && (pwq = kmem_cache_alloc(pwq_cache, GFP_KERNEL))) {
        /* 初始化等待队列, 指定ep_poll_callback为唤醒时的回调函数,
         * 当我们监听的fd发生状态改变时, 也就是队列头被唤醒时,
         * 指定的回调函数将会被调用. */
        init_waitqueue_func_entry(&pwq->wait, ep_poll_callback);  //  !!!!!!!!!!!!! 注册回调函数
        pwq->whead = whead;
        pwq->base = epi; //等待项的base为这个epitem节点

        /* 将刚分配的等待队列成员pwq加入到头中, 头是由fd持有的 */
        // pwq插入whead队列 whead由具体资源提供 比如文件 管道 资源满足条件时会pwd对应的回调函数 如现在的 ep_poll_callback
        // 插入EPOLLEXCLUSIVE解决惊群 一定程度上解决惊群问题
         if (epi->event.events & EPOLLEXCLUSIVE)
            add_wait_queue_exclusive(whead, &pwq->wait);
        else
            add_wait_queue(whead, &pwq->wait);

         // 插入关联的epi队列
        list_add_tail(&pwq->llink, &epi->pwqlist);
        /* nwait记录了当前epitem加入到了多少个等待队列中 */
        epi->nwait++;
    } else {
        /* We have to signal that an error occurred */
        epi->nwait = -1;
    }
}
```