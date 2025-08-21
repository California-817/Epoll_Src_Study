```c
//socket等待项回调函数, 当我们监听的fd发生状态改变时, 它会被调用.被内核ksoftirqd网络线程调用
static int ep_poll_callback(wait_queue_t *wait, unsigned mode, int sync, void *key)
{
    int pwake = 0;
    unsigned long flags;
    //从等待队列获取epitem.需要知道哪个进程挂载到这个设备
    struct epitem *epi = ep_item_from_wait(wait);
    struct eventpoll *ep = epi->ep;//获取epoll模型
    spin_lock_irqsave(&ep->lock, flags);

    if (!(epi->event.events & ~EP_PRIVATE_BITS)) //节点有EPOLLONESHOT选项  清除订阅的事件
        goto out_unlock;

    /* 没有我们关心的event... */
    if (key && !((unsigned long) key & epi->event.events))
        goto out_unlock;

    /* 
     * 这里看起来可能有点费解, 其实干的事情比较简单:
     * 如果该callback被调用的同时, epoll_wait()已经返回了,
     * 也就是说, 此刻应用程序有可能已经在循环获取events,
     * 这种情况下, 内核将此刻发生event的epitem用一个单独的链表
     * 链起来, 不发给应用程序, 也不丢弃, 而是在下一次epoll_wait
     * 时返回给用户.
     */
    if (unlikely(ep->ovflist != EP_UNACTIVE_PTR)) { //此时内核态用户线程清空rdlist到txlist中 ep_poll_callback不可以放节点到rdlist
        if (epi->next == EP_UNACTIVE_PTR) { //将就绪节点放到ovlist中
            epi->next = ep->ovflist;
            ep->ovflist = epi;
        }
        goto out_unlock;
    }

    /* 将当前的已经有就绪事件的epitem放入ready list */   节点既在红黑树中 又在rdlist中
    if (!ep_is_linked(&epi->rdllink)) 
        list_add_tail(&epi->rdllink, &ep->rdllist);

    /* 唤醒epoll_wait等待队列上的线程或进程... */
    if (waitqueue_active(&ep->wq))
        wake_up_locked(&ep->wq);
    /* 如果epollfd也在被poll, 那就唤醒队列里面的所有成员. */
    if (waitqueue_active(&ep->poll_wait))
        pwake++;

out_unlock:
    spin_unlock_irqrestore(&ep->lock, flags);

    /* We have to call this outside the lock */
    if (pwake)
        ep_poll_safewake(&ep->poll_wait);
    return 1;
}
```