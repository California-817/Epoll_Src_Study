```c
/* 这个函数真正将执行epoll_wait的进程带入睡眠状态... */
static int ep_poll(struct eventpoll *ep, struct epoll_event __user *events, //这个指针指向用户空间接受内核拷贝数据的地址
           int maxevents, long timeout) //超时时间
{
    int res, eavail;
    unsigned long flags;
    long jtimeout;
    wait_queue_t wait;//等待队列项

    /* 计算睡觉时间, 毫秒要转换为HZ */
    jtimeout = (timeout < 0 || timeout >= EP_MAX_MSTIMEO) ?
        MAX_SCHEDULE_TIMEOUT : (timeout * HZ + 999) / 1000;
retry:
    spin_lock_irqsave(&ep->lock, flags);
    res = 0;

    /* 如果ready list不为空, 就不睡了, 直接干活... */
    if (list_empty(&ep->rdllist)) {
        //rdlist队列为空
        /* OK, 初始化一个等待项, 准备直接把自己挂起,
         * 注意current是一个宏, 代表当前进程 */
        init_waitqueue_entry(&wait, current);//初始化等待队列项,这个等待项wait表示当前等待项 设置回调为autoremove_wake_function

        //当没有事件需要等待的时候，也需要把进程加入到等待队列中，加入到等待队列中的时候，也加了 EXCLUSIVE 标志
        __add_wait_queue_exclusive(&ep->wq, &wait);//挂载到ep结构的等待队列wq

        for (;;) {
            /* 将当前进程设置睡眠, 但是可以被信号唤醒的状态,
             * 注意这个设置是"将来时", 我们此刻还没睡! */
            set_current_state(TASK_INTERRUPTIBLE);
            /* 如果这个时候, ready list里面有成员了,   因为是死循环睡觉的
             * 或者睡眠时间已经过了, 就直接不睡了... */
            if (!list_empty(&ep->rdllist) || !jtimeout) 
                break;
            /* 如果有信号产生, 也起床... */
            if (signal_pending(current)) {
                res = -EINTR;
                break;
            }
            /* 啥事都没有,解锁, 睡觉... */
            spin_unlock_irqrestore(&ep->lock, flags);
            /* jtimeout这个时间后, 会被唤醒,
             * ep_poll_callback()如果此时被调用,
             * 那么我们就会直接被唤醒, 不用等时间了... 
             * 再次强调一下ep_poll_callback()的调用时机是由被监听的fd
             * 的具体实现, 比如socket或者某个设备驱动来决定的,
             * 因为等待队列头是他们持有的, epoll和当前进程
             * 只是单纯的等待...
             **/
            jtimeout = schedule_timeout(jtimeout);//真正挂起睡觉
            spin_lock_irqsave(&ep->lock, flags);}

        __remove_wait_queue(&ep->wq, &wait); //等待队列去除该等待项
        /* OK 我们醒来了... */
        set_current_state(TASK_RUNNING);
        }

    //rdlist队列不为空的时候直接走到这里 
    //或者之前没有事件通过ep_poll_callback回调 或者处理一次后rdlist非空唤醒 或者添加节点事件就绪时唤醒

    /* Is it worth to try to dig for events ? */
    eavail = !list_empty(&ep->rdllist) || ep->ovflist != EP_UNACTIVE_PTR;
    spin_unlock_irqrestore(&ep->lock, flags);

    /* 如果一切正常, 有event发生, 就开始ep_send_events准备数据copy给用户空间了... */
    if (!res && eavail &&
        !(res = ep_send_events(ep, events, maxevents)) && jtimeout) //当伪唤醒一个进程或线程 而事件已经被处理 
        goto retry; //线程或进程需要重新阻塞挂起等待---防止惊群现象产生
    return res; //返回就绪事件个数
}
```