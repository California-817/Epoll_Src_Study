```c
//当内核线程ksoftirqd接受网络数据包经过网络协议栈处理后 将数据放到socket的接受队列
//调用sk_data_ready来唤醒socket上的等待队列上的等待项，会调用等待项中设置的一个函数
//对于epoll设置进来的等待项 这个回调函数是ep_poll_callback函数
#define wake_up_locked_poll(x, m)                       \
    __wake_up_locked_key((x), TASK_NORMAL, poll_to_key(m))
//sk_data_ready会调用到这个函数__wake_up_locked_key
void __wake_up_locked_key(struct wait_queue_head *wq_head, unsigned int mode, void *key)
{
    __wake_up_common(wq_head, mode, 1, 0, key, NULL);
}

static int __wake_up_common(struct wait_queue_head *wq_head, unsigned int mode,
            int nr_exclusive, int wake_flags, void *key,
            wait_queue_entry_t *bookmark)
{
    wait_queue_entry_t *curr, *next;
    int cnt = 0;
    // 遍历socket的等待队列
    list_for_each_entry_safe_from(curr, next, &wq_head->head, entry) {
        unsigned flags = curr->flags;
        int ret;
        // 执行回调 对于epoll这个ep_poll_callback函数
        ret = curr->func(curr, mode, wake_flags, key); //执行完等待项的函数后并不会删除这个等待项
        if (ret < 0)
            break;
        // 设置了WQ_FLAG_EXCLUSIVE则只会回调一个等待项，nr_exclusive是1
        if (ret && (flags & WQ_FLAG_EXCLUSIVE) && !--nr_exclusive)
            break;
    }

    return nr_exclusive;
}
```