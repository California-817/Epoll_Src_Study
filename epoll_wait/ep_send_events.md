```c
//调用p_scan_ready_list()  拷贝内核空间的就绪事件到用户空间的events地址处
static int ep_send_events(struct eventpoll *ep,
              struct epoll_event __user *events, int maxevents)
{
    struct ep_send_events_data esed;
    esed.maxevents = maxevents;
    esed.events = events;
    return ep_scan_ready_list(ep, ep_send_events_proc, &esed); //涉及到两个函数哦
}
```