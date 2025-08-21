```c
//创建epoll模型接口
SYSCALL_DEFINE1(epoll_create, int, size)
{
   if(size <= 0）
      return -EINVAL;
      //这里会调用 epoll_create1中的函数
   return do_epoll_create(0);
}

static int do_epoll_create(int flags)
{
    int error, fd;
    struct eventpoll *ep = NULL;
    struct file *file;

    // 只支持CLOEXEC
    if (flags & ~EPOLL_CLOEXEC)
        return -EINVAL;

    // 分配一个eventpoll
    error = ep_alloc(&ep);

    // 获取一个空闲文件描述符
    fd = get_unused_fd_flags(O_RDWR | (flags & O_CLOEXEC));

    // 获取一个file，并且关联eventpoll_fops和上下文ep VFS
    file = anon_inode_getfile("[eventpoll]", &eventpoll_fops, ep, O_RDWR | (flags & O_CLOEXEC));

    // ep和file关联起来，上面是file和ep关联
    ep->file = file;
    // 关联fd和file 文件描述符数组fd下标处的指针指向这个file结构体对象
    fd_install(fd, file);
    return fd;
}
```