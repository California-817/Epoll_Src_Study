```c
// 创建号epollfd后，接下来添加fd
// epoll_ctl的参数：epfd表示epollfd；op有ADD，MOD，DEL
// fd是需要监听的描述符，event是我们感兴趣的events
SYSCALL_DEFINE4(epoll_ctl, int, epfd, int, op, int, fd, struct epoll_event __user *, event)
{
  int error;
  int did_lock_epmutex = 0;
  struct file *file, *tfile;
  struct eventpoll *ep;
  struct epitem *epi;
  struct epoll_event epds
  
  error = -EFAULT;
//错误处理以及从用户空间将epoll_event结构copy到内核空间.
    if (ep_op_has_event(op) &&  
            // 复制用户空间数据到内核 到内核栈上的epds
            copy_from_user(&epds, event, sizeof(struct epoll_event))) {  
        goto error_return;  
    }



  // 取得epfd对应的文件
  error = -EBADF;
  file = fget(epfd);  //fd的指针指向这个file
  if(!file) {
  goto error_return;
  }
  
  // 取得目标文件   epoll_ctl接口需要管理的那个socket的fd
  tfile = fget(fd); //获取到socket的文件结构体
  if(!tfile) {
  goto error_fput;
  }
  
    // 目标文件(socket)必须提供 poll 操作  
    error = -EPERM;  
    if (!tfile->f_op || !tfile->f_op->poll) {  
        goto error_tgt_fput;  
    }  

    // 添加自身或epfd 不是epoll 句柄  
    error = -EINVAL;  
    if (file == tfile || !is_file_epoll(file)) { //添加的事件不是自身 并且自身是epoll  
        goto error_tgt_fput;  
    }  

    // 取得内部结构eventpoll  
    ep = file->private_data; //这个指针指向epoll模型

    // EPOLL_CTL_MOD 不需要加全局锁(仅操作一个节点 不对整个数据结构修改) epmutex  
    if (op == EPOLL_CTL_ADD || op == EPOLL_CTL_DEL) {  
        mutex_lock(&epmutex);  
        did_lock_epmutex = 1;  
    }  
    
    //这一段逻辑是处理这个要管理的fd同样是一个epoll模型
    if (op == EPOLL_CTL_ADD) {  
        if (is_file_epoll(tfile)) {  
            error = -ELOOP;  
            // 目标文件也是epoll 检测是否有循环包含的问题  
            if (ep_loop_check(ep, tfile) != 0) {  
                goto error_tgt_fput;  
            }  
        } else  
        {  
            // 将目标文件添加到 epoll 全局的tfile_check_list 中  
            list_add(&tfile->f_tfile_llink, &tfile_check_list);  
        }  
    }  

    mutex_lock_nested(&ep->mtx, 0);  

    // 以tfile 和fd 为key 在rbtree 中查找文件对应的epitem  看节点是否已经存在 
    epi = ep_find(ep, tfile, fd);  

    error = -EINVAL; 
    //根据op类型进行不同的操作 
    switch (op) {  
    case EPOLL_CTL_ADD:   //添加节点
        if (!epi) {  
            // 没找到, 添加额外添加ERR HUP 事件  
            epds.events |= POLLERR | POLLHUP;  
            error = ep_insert(ep, &epds, tfile, fd);  
        } else {  
            error = -EEXIST;  //已经添加过了 返回错误 
        }  
        // 清空文件检查列表  
        clear_tfile_check_list();  
        break;  
    case EPOLL_CTL_DEL:  //删除节点
        if (epi) {  
            error = ep_remove(ep, epi);  
        } else {  
            error = -ENOENT;   //不存在还删除
        }  
        break;  
    case EPOLL_CTL_MOD:  //修改节点
        if (epi) {  
            epds.events |= POLLERR | POLLHUP;  
            error = ep_modify(ep, epi, &epds);  
        } else {   
            error = -ENOENT; //不存在还修改  
        }  
        break;  
    }  
    mutex_unlock(&ep->mtx);  

error_tgt_fput:  
    if (did_lock_epmutex) {  
        mutex_unlock(&epmutex);  
    }  

    fput(tfile);  
error_fput:  
    fput(file);  
error_return:  

    return error;  
}  
```