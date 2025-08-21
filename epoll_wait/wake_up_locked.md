```c
//epoll中在 ep_poll_callback中有就绪事件节点到rdlist中
//唤醒epoll中wq等待队列中一个等待线程或进程的函数
#define wake_up_locked(x)          __wake_up(x, TASK_NORMAL, 1, NULL)

void __wake_up(struct wait_queue_head *wq_head, unsigned int mode,
            int nr_exclusive, void *key)
{
    __wake_up_common_lock(wq_head, mode, nr_exclusive, 0, key);
}

static void __wake_up_common_lock(struct wait_queue_head *wq_head, unsigned int mode,
            int nr_exclusive, int wake_flags, void *key)
{
    unsigned long flags;
    wait_queue_entry_t bookmark;

    bookmark.flags = 0;
    bookmark.private = NULL;
    bookmark.func = NULL;
    INIT_LIST_HEAD(&bookmark.entry);
    // 这里只会执行一次
    do {
        spin_lock_irqsave(&wq_head->lock, flags);
        //核心逻辑__wake_up_common
        nr_exclusive = __wake_up_common(wq_head, mode, nr_exclusive,
                        wake_flags, key, &bookmark);
        spin_unlock_irqrestore(&wq_head->lock, flags);
    } while (bookmark.flags & WQ_FLAG_BOOKMARK);
}

static int __wake_up_common(struct wait_queue_head *wq_head, unsigned int mode,
            int nr_exclusive, int wake_flags, void *key,
            wait_queue_entry_t *bookmark)
{
    wait_queue_entry_t *curr, *next; 
    int cnt = 0;
    // 头指针所在结构体---curr就是wq等待队列上的一个等待项
    curr = list_first_entry(&wq_head->head, wait_queue_entry_t, entry);
    // 遍历队列----是epoll模型的等待队列wq
    list_for_each_entry_safe_from(curr, next, &wq_head->head, entry) {
        unsigned flags = curr->flags;
        int ret;
        // 执行回调 这个回调函数是autoremove_wake_function
        ret = curr->func(curr, mode, wake_flags, key);
        //自动设置这个选项 因为epoll_wait添加等待项的时候默认就是__add_wait_queue_exclusive方式放入
        // 设置了WQ_FLAG_EXCLUSIVE则只执行一次，即只唤醒一个等待项(进程/线程)，解决惊群问题
        if (ret && (flags & WQ_FLAG_EXCLUSIVE) && !--nr_exclusive)
            break;
    }
    return nr_exclusive;
}

//autoremove_wake_function流程很长 最终设置进程为就绪状态。
int autoremove_wake_function(struct wait_queue_entry *wq_entry, unsigned mode, int sync, void *key)
{
    int ret = default_wake_function(wq_entry, mode, sync, key); //唤醒进程或线程

    if (ret)
        list_del_init_careful(&wq_entry->entry); //删除epoll的wq等待队列上的该进程或线程等待项

    return ret;
}

int default_wake_function(wait_queue_entry_t *curr, unsigned mode, int wake_flags,
              void *key)
{
    WARN_ON_ONCE(IS_ENABLED(CONFIG_SCHED_DEBUG) && wake_flags & ~WF_SYNC);
    // curr->private为进程pcb即task_struct
    return try_to_wake_up(curr->private, mode, wake_flags);
}

static int
try_to_wake_up(struct task_struct *p, unsigned int state, int wake_flags)
{
    ttwu_runnable(p, wake_flags)；
}

static int ttwu_runnable(struct task_struct *p, int wake_flags)
{
    ttwu_do_wakeup(rq, p, wake_flags, &rf);
}

static void ttwu_do_wakeup(struct rq *rq, struct task_struct *p, int wake_flags,
               struct rq_flags *rf)
{
    p->state = TASK_RUNNING;  //唤醒线程或进程
}
```