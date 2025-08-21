# Epoll相关的几个问题
## 1.水平触发和边沿触发的区别
通俗来讲，水平触发，只要有数据，epoll_wait() 就会一直返回；边沿触发，如果数据没处理完，并且这个 fd 没有新的事件，那么再次 epoll_wait() 的时候也不会有事件上来。

水平触发还是边沿触发，影响的是接收事件的行为。

（1）举个例子

假如一个 tcp socket 一次收到了 1MByte 的数据，但是 epoll_wait(） 返回之后，用户一次只读取 1KB 的数据，读完之后再进行 epoll_wait() 监听事件，示例代码如下所示：
```cpp
    for (;;) {
        event_num = epoll_wait(epoll_fd, events, 1, -1);
        ret = recv(tcp_fd, buff, 1024, 0);
        if (ret <= 0) {
            break;
       }
    }
```
如果是水平触发，那么每次 epoll_wait() 都能返回这个 tcp_fd 的事件，共返回 （1MB / 1KB）= 1024 次事件；如果是边沿触发，那么只有第一次返回了 tcp_fd 的事件，后边的 epoll_wait() 也不返回了。

（2）epoll源码分析

从代码实现来理解边沿触发和水平触发之间的区别。
用户在调用 epoll_wait() 之后，如果有事件，最终会调用 ep_send_events() 来做事件的转移，
主要工作是将事件从 rdllist 中转移到用户的 events 数组。
```CPP
static int ep_send_events(struct eventpoll *ep,
              struct epoll_event __user *events, int maxevents)
{
    ep_start_scan(ep, &txlist);
    list_for_each_entry_safe(epi, tmp, &txlist, rdllink)
        revents = ep_item_poll(epi, &pt, 1);
        if (!revents)
            continue;
 
        events = epoll_put_uevent(revents, epi->event.data, events);
    }
    res++;
    if (epi->event.events & EPOLLONESHOT)
        epi->event.events &= EP_PRIVATE_BITS;
    else if (!(epi->event.events & EPOLLET)) { // 如果不是边沿触发，则将 epitem 重新加回就绪链表
        list_add_tail(&epi->rdllink, &ep->rdllist);
        ep_pm_stay_awake(epi);
    }
    return res;
}
```
对于从 rdllist 中获取的 epitem，首先调用 ep_item_poll(） 确定有没有事件，以及事件的具体类型，如果有事件则件通过 epoll_put_uevent() 将事件保存存到用户的 event 数组中。

之后会判断是否为边沿触发，如果不是边沿触发，即水平触发，还会把这个 epitem 加入到 rdllist 的尾部，这样下次 epoll_wait() 的时候还会 poll 这个 fd，还会对这个 fd 进行 poll 操作，如果有事件还会向用户返回。

以上边举的例子来看，第 1 到 1024 次调用 epoll_wait() 都会返回事件，第 1025 次调用 epoll_wait() 的时候，由于数据都已经读取完毕，所以通过 ep_item_poll() 发现没有事件，这次当然也不会向用户返回事件，同时也不会把 epitem 返回 rdllist 了。

（3）实验
```CPP
server:

#include <stdio.h>
#include <string.h>
#include <pthread.h>
#include <unistd.h>
#include <sys/socket.h>
#include <sys/epoll.h>
#include <netinet/in.h>
#include <arpa/inet.h>
#include <fcntl.h>
#include <errno.h>
 
#define SERVER_IP     ("0.0.0.0")
#define SERVER_PORT   (12345)
#define MAX_LISTENQ   (32)
#define MAX_EVENT     (128)
 
int create_tcp_server() {
    int listen_fd = socket(AF_INET, SOCK_STREAM, 0);
    if (listen_fd < 0) {
        printf("create socket error: %s\n", strerror(errno));
        return -1;
    }
 
    struct sockaddr_in server_addr;
    memset(&server_addr, 0, sizeof(server_addr));
    server_addr.sin_family = AF_INET;
    server_addr.sin_addr.s_addr = inet_addr(SERVER_IP); /**< 0.0.0.0 all local ip */
    server_addr.sin_port = htons(SERVER_PORT);
 
    if (bind(listen_fd,(struct sockaddr *)&server_addr, sizeof(server_addr)) < 0) {
        printf("bind[%s:%d] error.\n", SERVER_IP, SERVER_PORT);
        return -1;
    }
 
    if (listen(listen_fd, MAX_LISTENQ) < 0) {
        printf("listen error.\n");
        return -1;
    }
 
    return listen_fd;
}
 
int main() {
    int ret = -1;
    int sock_fd = -1;
    int accetp_fd = -1;
    int event_num = -1;
    int epoll_fd  = -1;
    int listen_fd = -1;
 
    struct sockaddr_in client_addr;
    struct sockaddr_in server_addr;
    socklen_t client = sizeof(struct sockaddr_in);
 
    struct epoll_event ev;
    struct epoll_event events[MAX_EVENT];
 
    listen_fd = create_tcp_server();
    if (listen_fd < 0) {
        printf("create server error\n");
        return -1;
    }
 
    epoll_fd = epoll_create(MAX_EVENT);
    if (epoll_fd <= 0) {
        printf("cteare epoll failed, error: %s\n", strerror(errno));
        return -1;
    }
 
    ev.data.fd = listen_fd;
    ev.events = EPOLLIN;
    if (epoll_ctl(epoll_fd, EPOLL_CTL_ADD, listen_fd, &ev) < 0) {
        printf("add listen fd to epoll error.\n");
        return -1;
    }
 
    char buff[1024] = {'\0'};
    int seq = 0;
    for (;;) {
        printf("epoll wait:\n");
        event_num = epoll_wait(epoll_fd, events, MAX_EVENT, -1);
        printf("events num:%d\n", event_num);
        for (int i = 0; i < event_num; i++ ) {
            if (events[i].events & EPOLLIN == 0) {
                /* 只监听 EPOLLIN 事件 */
                printf("fd %d error, cloase it, event[0x%x].\n", events[i].data.fd, events[i].events);
                close(events[i].data.fd);
                return -1;
            }
            
            if (events[i].data.fd == listen_fd) {
                accetp_fd = accept(listen_fd, (struct sockaddr*)&client_addr, &client);
                if(accetp_fd < 0) {
                    printf("accept error.\n");
                    return -1;
                }
 
                ev.data.fd = accetp_fd;
                ev.events = EPOLLIN; // 水平触发，默认触发方式
                // ev.events = EPOLLIN | EPOLLET; // 边沿触发，需要通过标记 EPOLLET 来指定
                if(epoll_ctl(epoll_fd, EPOLL_CTL_ADD, accetp_fd, &ev) < 0) {
                    printf("add fd to epoll error.\n");
                    return -1;
                }
                sleep(2); // 之所以等 2 秒，是为了让客户端发送 1M 的数据发送完毕
            } else {
                ret = recv(accetp_fd, buff, 1024, 0);
                if (ret <= 0) {
                    printf("recv error.\n");
                    epoll_ctl(epoll_fd, EPOLL_CTL_DEL, accetp_fd, &ev);
                    close(accetp_fd);
                    events[i].data.fd = -1;
                    return -1;
                }
                printf("seq: %d, recv len: %d\n", seq, ret);
                seq++;
            }
        }
    }
 
    close(listen_fd);
}
```
```CPP
client:

#include <stdio.h>
#include <string.h>
#include <stdlib.h>
#include <errno.h>
#include <fcntl.h>
#include <sys/epoll.h>
#include <sys/types.h>
#include <sys/socket.h>
#include <unistd.h>
#include <netinet/in.h>
#include <arpa/inet.h>
 
#define SERVER_PORT   (12345)
#define SERVER_IP     "127.0.0.1"
#define MAX_EVENT     (128)
#define MAX_BUFSIZE   (512)
 
int main(int argc,char *argv[]) {
    int sock_fd;
    sock_fd = socket(AF_INET, SOCK_STREAM, 0);
    if(sock_fd < 0) {
        printf("create socket error.\n");
        return -1;
    }
 
    struct sockaddr_in addr_serv;
    memset(&addr_serv, 0, sizeof(addr_serv));
 
    addr_serv.sin_family = AF_INET;
    addr_serv.sin_port =  htons(SERVER_PORT);
    addr_serv.sin_addr.s_addr = inet_addr(SERVER_IP);
 
    if(connect(sock_fd, (struct sockaddr *)&addr_serv,sizeof(struct sockaddr)) < 0){
        printf("connect error.\n");
        return -1;
    }
    char send_buff[1024 * 1024] = "epoll test!";
    int send_total_count = 0;
    int send_tmp_count = 0;
    while (send_total_count < 1024 * 1024) {
        send_tmp_count = send(sock_fd, send_buff, 1024 * 1024, 0);
        send_total_count += send_tmp_count;
    }
    printf("send 1MB success\n");
 
    close(sock_fd);
    return 0;
}
```
水平触发测试结果：

计数从 0 开始，到 1023 结束，共接收 1024 次数据，共 1MB


边沿触发测试结果：

第一次接受 1024 的数据之后，后边 epoll_wait() 不会返回事件


## 2.epoll 惊群问题是怎么回事
（1）现象

在 linux 网络方面讨论惊群问题的时候，通常有两个惊群问题，分别是 accept 惊群问题和 epoll 惊群问题。如果多个进程阻塞到同一个 fd 上，当这个 fd 有事件的时候，多个进程会被唤醒，但是只有一个进程获得并处理该事件，导致其它进程空转，进而导致 cpu 资源的浪费。这就是惊群问题。

如果你见过养鸡的，那么很好理解惊群问题。假设养了一群鸡，这个时候你要喂鸡，鸡群看到你来了，都跑了过来，但是这个时候，你手中只有一粒玉米，你往空中一抛，所有的鸡都跳了起来，但是只有一只鸡能吃到这粒玉米，其它集白跳了一次，这就是惊群问题。


从 2.6 开始，linux 内核已经解决了 accept 惊群问题， 具体的解决方式就是给等待队列中的元素增加一个 EXCLUSIVE(排他)标记。


当多个进程阻塞在同一个 listening fd 上时，进程就会被加入到这个 fd 的等待队列中，当 fd 中有事件到来的时候，就会唤醒等待队列中的进程。当没有 EXCLUSIVE 标记的时候，等待队列中的元素全部会被唤醒。

后来给等待元素增加了 EXCLUSIVE 标记，并且带此标记的进程会加到等待队列的尾部。唤醒的时候，从队列头开始唤醒，没有 EXCLUSIVE 标记的进程都会被唤醒，有这个标记的进程只唤醒一个，这样就保证了 EXCLUSIVE 的进程只唤醒一个。

当用户调用 accept()，在内核态会调用到 inet_csk_accept()，如果当前没有新的连接到来，那么就会调用 inet_csk_wait_for_connet()， 进而调用到 prepare_to_wait_exclusive()，从这个函数的名字来看，就是排他性的。所以 accept() 通过 EXCLUSIVE 标志解决了惊群问题。
```cpp
struct sock *inet_csk_accept(struct sock *sk, int flags, int *err, bool kern)
{
    if (reqsk_queue_empty(queue)) {
        error = inet_csk_wait_for_connect(sk, timeo);
        if (error)
            goto out_err;
    }
}
 
static int inet_csk_wait_for_connect(struct sock *sk, long timeo)
{
    for (;;) {
        prepare_to_wait_exclusive(sk_sleep(sk), &wait,
                      TASK_INTERRUPTIBLE);
    }
}
```
（2）原因

由上边分析可知，解决 accept() 惊群问题就是在把进程加入到等待队列时，增加了 EXCLUSIVE 标志，并且唤醒的时候，对于有该标志的进程只唤醒一个。

而在 epoll_wait() 的内核实现中，当没有事件需要等待的时候，也需要把进程加入到等待队列中，加入到等待队列中的时候，也加了 EXCLUSIVE 标志，为什么这种方式下 epoll 仍然有惊群问题呢 ？
```cpp
static int ep_poll(struct eventpoll *ep, struct epoll_event __user *events,
           int maxevents, struct timespec64 *timeout)
{
    while (1) {
        if (!eavail)
            __add_wait_queue_exclusive(&ep->wq, &wait);
        }
    }
}
```
可以说，通过 EXCLUSIVE 标志，内核在一定程度上解决了 epoll 惊群问题，但是没有完全解决。

先通过一个例子看一下 epoll 惊群的现象，然后再结合代码分析产生这种现象的原因：
```cpp
#include <stdlib.h>
#include <stdio.h>
#include <string.h>
#include <unistd.h>
#include <fcntl.h>
#include <netdb.h>
#include <sys/types.h>
#include <sys/socket.h>
#include <sys/epoll.h>
#include <sys/wait.h>
#include <errno.h>
 
 
#define SERVER_IP     ("0.0.0.0")
#define SERVER_PORT   (12345)
#define MAX_LISTENQ   (32)
#define MAX_EVENT     (128)
#define PROCESS_NUM   (10)
 
int set_fd_nonblocking(int fd){  
    int val = fcntl(fd, F_GETFL);  
    val |= O_NONBLOCK;  
    if(fcntl(fd, F_SETFL, val) < 0){  
        return -1;  
    }  
    return 0;  
}
 
int set_tcp_server_reuse(int fd) {
    int so_reuse = 1;
 
    if (setsockopt(fd, SOL_SOCKET, SO_REUSEADDR, &so_reuse, sizeof(so_reuse)) < 0) {
        printf("fd[%d] set SO_REUSEADDR error[%s]", fd, strerror(errno));
        return -1;
    }
    return 0;
}
 
int create_tcp_server() {
    int listen_fd = socket(AF_INET, SOCK_STREAM, 0);
    if (listen_fd < 0) {
        printf("create socket error: %s\n", strerror(errno));
        return -1;
    }
    set_tcp_server_reuse(listen_fd);
 
    struct sockaddr_in server_addr;
    memset(&server_addr, 0, sizeof(server_addr));
    server_addr.sin_family = AF_INET;
    server_addr.sin_addr.s_addr = inet_addr(SERVER_IP); /**< 0.0.0.0 all local ip */
    server_addr.sin_port = htons(SERVER_PORT);
 
    if (bind(listen_fd,(struct sockaddr *)&server_addr, sizeof(server_addr)) < 0) {
        printf("bind[%s:%d] error: %s.\n", SERVER_IP, SERVER_PORT, strerror(errno));
        return -1;
    }
 
    if (listen(listen_fd, MAX_LISTENQ) < 0) {
        printf("listen error.\n");
        return -1;
    }
 
    return listen_fd;
}
 
int main() {
    int ret = -1;
    int accept_fd = -1;
    int event_num = -1;
    int epoll_fd  = -1;
    int listen_fd = -1;
 
    struct sockaddr_in client_addr;
    struct sockaddr_in server_addr;
    socklen_t client = sizeof(struct sockaddr_in);
 
    struct epoll_event ev;
    struct epoll_event events[MAX_EVENT];
 
    listen_fd = create_tcp_server();
    if (listen_fd < 0) {
        printf("create server error\n");
        return -1;
    }
    set_fd_nonblocking(listen_fd);
 
    epoll_fd = epoll_create(MAX_EVENT);
    if (epoll_fd <= 0) {
        printf("cteare epoll failed, error: %s\n", strerror(errno));
        return -1;
    }
 
    ev.data.fd = listen_fd;
    ev.events = EPOLLIN;
    if (epoll_ctl(epoll_fd, EPOLL_CTL_ADD, listen_fd, &ev) < 0) {
        printf("add listen fd to epoll error.\n");
        return -1;
    }
 
    for(int i = 0; i < PROCESS_NUM; i++){
        int pid = fork();
        if(pid == 0){
            while(1){
                int event_num = 0;
                event_num = epoll_wait(epoll_fd, events, MAX_EVENT, -1);
                printf("process %d return from epoll_wait, event num: %d\n", getpid(), event_num);
                sleep(1);
                for(i = 0; i < event_num; i++){
                    if(!(events[i].events & EPOLLIN)){
                        printf("epoll error\n");
                        return -1;
                    }else if(listen_fd == events[i].data.fd){
                        struct sockaddr in_addr;
                        socklen_t in_len = sizeof(in_addr);
                        accept_fd = accept(listen_fd, &in_addr, &in_len);
                        printf("process %d accept fd %d\n", getpid(), accept_fd);
                        if(accept_fd <= 0){
                            printf("process %d accept failed!\n", getpid());
                        }else{
                            printf("process %d accept successful!\n", getpid());
                        }
                    }
                }
            }
        }
    }
    wait(0);
    return 0;
}
```
代码解释：

上述代码，首先在主进程中创建一个 epoll fd, 并加入一个 tcp listening fd，然后创建 10 个子进程，这 10 个子进程均对主进程创建的 epoll fd 进行 epoll wait。

设置 socket 选项 SO_REUSEADDR 是为了避免在测试过程中出现 “Address already in use.” 错误。

将 fd 设置为非阻塞，是为了在没有新连接的时候也让 accept() 返回，防止其中一个进程将新连接接收之后，其它进程一直阻塞在 accept() 系统调用，看不到后边的接收成功与否的打印。

测试过程：

上述代码编译之后，在一个终端启动，在另外一个终端通过 telnet 链接监听的端口，telnet 127.0.0.1 12345。

测试现象：

基于上边代码和测试步骤，做三个不同的实验，这三个实验呈现不同的现象。

这三个实验的区别有两方面，分别是监听 fd 是水平触发还是边沿触发， epoll_wait() 返回之后要不要 sleep(）。

### 实验 一：水平触发 + epoll_wait() 返回之后 sleep()

10 个进程都被唤醒，其中一个进程接收了新连接。出现了惊群现象。


### 实验 二：水平触发 + epoll_wait() 返回之后不进行 sleep()

被唤醒进程的个数不确定， 范围在 [1, 10] 之间，其中一个进程接收了新连接。出现了惊群现象，但是没有实验 一那么严重。


### 实验 三：边沿触发 + epoll_wait() 返回之后 sleep()

只有一个进程被唤醒，该进程接收了新链接。没有出现惊群现象。


下边通过分析源码，来解释上边三个现象：

在 epoll_wait() 函数中，如果有事件，那么就会唤醒等待进程。进程会调用函数 ep_send_events() 将事件从就绪链表转移到用户态的 event 数组。该函数最后调用了 ep_done_scan()，这个函数最后判断了就绪队列是不是空，如果非空的话，会继续唤醒等待队列中的进程。唤醒一个等待进程之后，当前这个进程就会返回用户态。
```cpp
static int ep_send_events(struct eventpoll *ep,
              struct epoll_event __user *events, int maxevents)
{
    ep_done_scan(ep, &txlist);
}
 
static void ep_done_scan(struct eventpoll *ep,
             struct list_head *txlist)
{
    if (!list_empty(&ep->rdllist)) {
        if (waitqueue_active(&ep->wq))
            wake_up(&ep->wq);
    }
 
    write_unlock_irq(&ep->lock);
}
```
而后边被唤醒的进程，当被执行的时候，有两种情况：

① 事件已经被之前被唤醒的进程处理，那么该进程检查没有事件，继续睡眠等待，不会返回用户态；

② 事件还没有被处理，那么该进程的工作流程和第一个被唤醒的进程相同，会返回用户态。

根据上边两种情况可以解释实验一和实验二的区别，实验一，在 epoll_wait() 返回之后进行 sleep()，就是为了不让返回的进程立即处理事件，这样就会导致所有的进程都被唤醒，返回用户态；实验二，epoll_wait() 返回之后会立即处理事件，这就和操作系统调度的快慢有关系了，被唤醒的进程个数是不确定的。

当就绪链表中有事件的时候，就会继续唤醒进程，如果是边沿触发的话，也不会惊群了，因为一次之后这个就不会加回就绪链表了。

实验一和实验三之间的区别就是水平触发和边沿触发的区别，因为水平触发的话，事件转移到用户态之后，还会加回就绪队列，这样导致就绪队列不为空，进而导致会继续唤醒进程；而边沿触发不会把事件加回就绪队列，这就绪队列为空，所以不会继续唤醒进程

### （3）epoll 惊群如何规避

从上边的分析来看，linux 内核完全解决了 accept() 的惊群问题，因为对于 accept() 来说，一个连接事件是原子的，不能拆分的，只能被一个进程处理，所以内核这么实现是没有问题的，也不会引起用户的异议。

但是 epoll 就不一样，因为 epoll 除了能监听连接事件以外还可以监听接收，发送等事件。就拿接收事件来说，用户可能希望一次接收的数据用多个进程或线程来处理，如果内核强制成只唤醒一个进程，那么就满足不了用户的需求了。

所以说 epoll 惊群，严格意义上来说，并不是问题，而是将最终的决策权交给了用户。

## 3.为什么当发送返回失败时才需要监听 EPOLLOUT 事件
（1）接收数据和发送数据的区别: 一个被动，一个主动

使用一个连接(tcp socket) 无非就是做两件事情，接收数据和发送数据，

对于接收数据来说，接收方是被动的，只有对端的数据到达本端之后，才会产生 EPOLLIN 事件，之后进行实际数据的接收才是有意义的；

而对于发送来说，发送端是主动的，不是被动的，也就是说只要发送缓冲区有充足的空间，那么发送端就可以发送数据，而不需要有 EPOLLOUT 事件才能发送数据。

所以对于发送数据来说，并不需要一开始就监听 EPOLLOUT 事件，只需要等到这次发送失败（实际发送的数据长度小于要发送的数据长度), 才需要监听发送事件。而一旦 EPOLLOUT 事件上来之后，就需要清除这个事件，不需要一直监听，否则的话每次 epoll wait，只要发送侧有缓冲区可用，都会返回可写事件。

（2）tcp 上报 EPOLLOUT 事件的判断条件
```cpp
static inline bool __sk_stream_is_writeable(const struct sock *sk, int wake)
{
        // 条件1：剩余空间大于发送缓冲区最大值的 1/3
        // 条件2：沒有发送出去的要小于 lowwat 的临界值
    return sk_stream_wspace(sk) >= sk_stream_min_wspace(sk) &&
           __sk_stream_memory_free(sk, wake);
}
```
## 4.epoll 相对于 select, poll 有什么区别
### （1）epoll 在内核态和用户态，只需要遍历有事件的 fd，不用全部遍历

这个是 epoll 和 select， poll 最明显的别，也是 epoll 最明显的优势。

假如要监听 500 个 fd，而现在只有 10 个 fd 有事件，对于 select 和 poll 来说，当调用 select(), poll() 之后，内核需要把 500 个 fd 遍历一遍，然后返回，返回之后，用户处理事件的时候，仍然要把所有 fd 都遍历一遍，同时处理有事件的 fd。

对于 epoll 来说，当调用 epoll_wait() 之后，在内核态只需要遍历 10 个文件描述符，返回之后，在用户态也只需要处理 10 个事件即可，不需要遍历全部的 500 个描述符。


由此可见，select 和 poll 适用于 fd 个数比较少，或者 fd 个数比较多并且大多数 fd 都比较活跃的场景；而 epoll 适用于 fd 个数比较多，但是活跃 fd 占比不大的场景。

### （2）监听的文件个数限制

① select 只能监听 1024 个文件描述符（0 ~ 1023）

select 能监听的 fd 的个数是有限制的，1024 个，并且这 1024 个 fd 的范围也是有限制的，只能是 0~ 1023, 假如你创建了一个 fd 是 2000， 但是被 select 监听的 fd 只有 10 个(没有超过数量限制)，那么这个 2000 的 fd 也是不能被 select 监听。因为 fd_set 在内核是一个 bitmap, 用 bitmap 的位来表示一个 fd，所以能表示的 fd 的范围就是 0 ~ 1023。
```c
int select(int nfds, fd_set *readfds, fd_set *writefds,
                  fd_set *exceptfds, struct timeval *timeout);
 
#define __FD_SETSIZE    1024
typedef struct {
    unsigned long fds_bits[__FD_SETSIZE / (8 * sizeof(long))];
} __kernel_fd_set;
```
② poll 和 epoll 本身没有对文件按描述符个数的限制。

### （3）目标事件和实际事件分离

① select 目标事件和实际事件没有分离

select 每次调用之前，都需要将自己要监听的读事件，写事件，错误事件分别设置到对应的 fd_set 中，内核处理的时候，如果要监听的事件，当前没有，就会把对应位置清除，返回给用户态。所以对于 select 来说，用户要监听的事件，以及用户返回的事件，都放到 fd_set 里边，共用这块数据空间。这就导致每次调用 select 时，都需要重新设置一边 fd_set。

② poll 和 epoll 实现了分离

poll 使用一个结构体 strucy poll_fd 来表示一个 被监听的 fd，并且目标事件和实际事件通过 events 和 reevents 来表示，这样对于用户来说不需要每次调用 poll() 之前都设置一遍目标事件。
```c
struct pollfd {
    int fd;
    short int events;
    short int revents;
};
```
epoll 通过一个专门的 api epoll_ctl 来管理要监听的目标事件，epoll_wait() 的时候直接获取数据事件。

### （4）系统调用个数的区别

select 和 poll 均是只有一个 系统调用，这样使用简单，但是每次都要传递要监听的 fd 以及监听的目标事件，导致在用户态和内核态传递的数据量较大；

epoll 通过三个 api 来实现，epoll_create(), epoll_ctl(), epoll_wait()。epoll 本身也占用一个 fd 资源，select，poll 不存在这种情况，但是 epoll 通过这种方式，将控制面和数据面完全分离，要监听哪些 fd，以及每个 fd 要监听的目标事件，完全在内核进行管理，这就减少了 epoll_wait() 时在用户态和内核态传输的数据量