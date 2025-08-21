```c
//由ep_send_events()调用本函数 拷贝内核空间的就绪事件到用户空间
static int ep_scan_ready_list(struct eventpoll *ep,
                  int (*sproc)(struct eventpoll *,  // 这个函数是 ep_send_events_proc
                       struct list_head *, void *),
                  void *priv)  //这个priv指针指向的对象内部存储了用户空间的地址
{
    int error, pwake = 0;
    unsigned long flags;
    struct epitem *epi, *nepi;
    LIST_HEAD(txlist);
    mutex_lock(&ep->mtx);

    spin_lock_irqsave(&ep->lock, flags); //加锁
    
    /* 这一步要注意, 首先, 所有监听到events的epitem都链到rdllist上了,
     * 但是这一步之后, 所有的epitem都转移到了txlist上, 而rdllist被清空了,
     * 要注意哦, rdllist已经被清空了! */
    list_splice_init(&ep->rdllist, &txlist);

    /* ovflist, 在ep_poll_callback()里面我解释过, 此时此刻我们不希望
     * 有新的event加入到ready list中了, 保存后下次再处理... */
    ep->ovflist = NULL;  //暂时存储此时触发的事件
    spin_unlock_irqrestore(&ep->lock, flags);

    /* 在这个回调函数里面处理每个epitem
     * sproc 就是 ep_send_events_proc, 下面会注释到. */
    error = (*sproc)(ep, &txlist, priv);  //txlist中才是所有就绪节点的存放地点

    spin_lock_irqsave(&ep->lock, flags);

    /* 现在我们来处理ovflist, 这些epitem都是我们在传递数据给用户空间时
     * 监听到了事件. */
    for (nepi = ep->ovflist; (epi = nepi) != NULL;
         nepi = epi->next, epi->next = EP_UNACTIVE_PTR) {

        /* 将这些直接放入readylist */
        if (!ep_is_linked(&epi->rdllink))
            list_add_tail(&epi->rdllink, &ep->rdllist);
    }

    ep->ovflist = EP_UNACTIVE_PTR; //没有设置这个选项时 就绪事件时不能放到rdlist中 只可以放到ovlist中

    /*
       把剩下的移到就绪队列 ep_read_events_proc里面会移除txlist列表的节点
       但是可能因为达到阈值 没有处理完。见ep_read_events_proc里面的
       esed->res >= esed->maxevents逻辑
       上一次没有处理完的epitem, 重新插入到ready list */
    list_splice(&txlist, &ep->rdllist);

    //此时如果这个rdlist不为空 节点来源有三个来源:  1.ovlist中的节点 2.txlist中未处理完的点 3.LT触发模式节点会重新放回rdlist

    /* ready list不为空, 直接唤醒线程或进程... */
    if (!list_empty(&ep->rdllist)) {
        if (waitqueue_active(&ep->wq))
            wake_up_locked(&ep->wq);
        if (waitqueue_active(&ep->poll_wait))
            pwake++;
    }

    spin_unlock_irqrestore(&ep->lock, flags);
    mutex_unlock(&ep->mtx);
    /* We have to call this outside the lock */
    if (pwake)
        ep_poll_safewake(&ep->poll_wait);
    return error;
}
```