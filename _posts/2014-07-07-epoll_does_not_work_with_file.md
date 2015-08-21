---
layout: post
title:  "普通文件为什么不能使用epoll"
date:   2014-07-07
categories: linux kernel
---

##问题引出
在FreeBSD下的 `kqueue` 可以用来监控普通文件的变化, 可参考 [developer.apple.com](https://developer.apple.com/library/mac/documentation/Darwin/Conceptual/FSEvents_ProgGuide/KernelQueues/KernelQueues.html) 的说明.  
自然想到Linux下类似功能的 `epoll`  
尝试写一个小程序:

```
#include <stdio.h>
#include <sys/epoll.h>
#include <fcntl.h>

int main()
{
    int epfd = epoll_create(5);
    if (epfd < 0) {
        perror("create epoll");
        return -1;
    }
    int fd = open("watch.txt", O_RDONLY);
    if (fd == -1) {
        perror("open file");    
        return -1;
    }

    struct epoll_event ev;
    ev.events = EPOLLIN;
    int rc = epoll_ctl(epfd, EPOLL_CTL_ADD, fd, &ev);    
    if (rc < 0) {
        perror("epoll ctl add");
        return -1;
    }

    struct epoll_event events[2];
    epoll_wait(epfd, events, 2, -1);
    printf("file changed!\n");

    return 0;
}
```

编译运行一下:

```
[root@localhost epoll]# gcc main.c    
[root@localhost epoll]# ./a.out 
epoll ctl add: Operation not permitted
```

出错了, 很直观的猜测原因:   
` man epoll_wait`
>The  epoll_wait()  system  call  waits  for events on the epoll instance referred to by the file descriptor epfd.

`man epoll_ctl`
>EPOLLIN: The associated file is available for read(2) operations.

很简单, 普通文件在任何情况下都是可写的, 压根不需要 epoll 来 "wait" 嘛.   
另, `epoll_ctl` 的失败时, 返回的是 `EPERM`, 意思是:
>The target file fd does not support epoll.

那就顺着这个找原因
##为什么不支持?
###epoll实现
源码位于 [fs/eventpoll.c](http://lxr.free-electrons.com/source/fs/eventpoll.c) 

查看 `epoll_ctl` 实现中有如下流程:

```
SYSCALL_DEFINE4(epoll_ctl, int, epfd, int, op, int, fd,
        struct epoll_event __user *, event)
{
    ...
    /* Get the "struct file *" for the target file */
    tfile = fget(fd);
    if (!tfile)
        goto error_fput;

    /* The target file descriptor must support poll */
    error = -EPERM;
    if (!tfile->f_op || !tfile->f_op->poll)
        goto error_tgt_fput;
```

看到这里会检查 file 结构体中是否配置了 `poll` 函数, 如果没有, 就会返回 `EPERM` .

###文件系统相关
我的实验环境是:

```
[root@localhost epoll]# file -s /dev/sda5 
/dev/sda5: Linux rev 1.0 ext3 filesystem data (needs journal recovery) (large files)
```

查看 ext3 的实现, 位于 [fs/ext3/file.c](http://lxr.free-electrons.com/source/fs/ext3/file.c) ,看到

```
const struct file_operations ext3_file_operations = {
    .llseek     = generic_file_llseek,
    .read       = do_sync_read,
    .write      = do_sync_write,
    .aio_read   = generic_file_aio_read,
    .aio_write  = generic_file_aio_write,
    .unlocked_ioctl = ext3_ioctl,
#ifdef CONFIG_COMPAT
    .compat_ioctl   = ext3_compat_ioctl,
#endif
    .mmap       = generic_file_mmap,
    .open       = generic_file_open,
    .release    = ext3_release_file,
    .fsync      = ext3_sync_file,
    .splice_read    = generic_file_splice_read,
    .splice_write   = generic_file_splice_write,
};
```
file 结构体中 `f_op` (struct file\_operations) 的赋值中没有 poll 函数的赋值, 所以造成了 epoll 中获取到的 tfile->f\_op->poll 为空, 那就是这了.  
说明 ext3 下的文件是不支持 poll 的, 至于其他的, 也是拿这个标准确定.

另, `poll` 和 `select` 也是调用的这个 poll 函数

##什么可以使用epoll
根据前面的分析, 有 poll 函数才能支持 epoll, 在 Linux 源码中的 fs 下的源码中搜索 `\.poll\s*=` , 看到以下这些文件系统都支持 poll:

* eventfd
* eventpoll
* pipe
* signalfd
* timerfd
* kmsg

等等, 大部分我都没有接触过, 平时比较常用的也就是 `socket` 了.

