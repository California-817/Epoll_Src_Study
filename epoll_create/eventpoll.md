```c
//epoll的核心实现对应于一个epoll描述符
//这个epoll模型是通过虚拟文件系统VFS,将 struct file 的private data指针指向这个模型 从而在上层通过fd访问这个epoll模型

struct eventpoll{
  spinlock_t lock;
  struct mutex mtx;
  wait_queue_head_t wq; // sys_epoll_wait() 在这里等待  （进程或线程epoll_wait无就绪节点时 会在这个队列上等待挂起）
  
  // 当epoll被另一个epoll监听时需要使用poll_wait记录阻塞在该epoll的队列
  wait_queue_head_t poll_wait; //f_op->poll() 使用的，被其他事件通知机制使用的wait_address

  struct list_head rdlist; //存储已经就绪的epitem的列表
  
  struct rb_root rbr; //保存fd对应的epitem，并且以红黑树的形式组织起来
  
  struct epitem *ovlist; //当内核向用户空间复制数据时，发生的活跃事件，暂时存储起来
  
  struct user_struct *user; //该epollfd的使用者用户
  
  struct file *file; //epollfd对用的内核文件
  
  int visited; //加快检测效率
  
  struct list_head visited_list_link
}
```
