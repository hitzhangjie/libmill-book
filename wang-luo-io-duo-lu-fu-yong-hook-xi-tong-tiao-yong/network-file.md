---
description: >-
  libmill是一个基于c语言开发的go风格协程库，实现了go风格的协程操作、chan操作、非阻塞的网络io操作等等，是一个不错的linux平台下c风格协程库实现。如果你想从0到1的快速了解如何开发一个协程库，libmill将是一个非常不错的案例；如果你想更深入地了解go，libmill里面也借鉴了go的一些设计思想；或者你想在生产环境中使用，开发者也提供了一个更健壮的版本libdill。
---

# network: file

#### file

epoll并不支持所有的fd类型，一般将epoll应用于socket、pipe、tty以及其他有限的设备类型，epoll不支持regular file。尽管我们可以传递regular file的fd给select、poll但这只是因为select、poll在接口上允许而已，并没有什么效果（总是返回事件就绪\)，既然没有什么效果，epoll接口在设计的时候就决定根本不接受regular file fd，实际上epoll也只是为了改善select、poll并没打算额外支持regular file。这里只是将epoll应用在/dev/pts设备上，stdin、stdout、stderr都事这种设备类型，并不是针对regular file的。

难道os就无法支持regular file的io多路复用吗？怎么可能不能？只是epoll不支持，bsd下kqueue就支持！

这里的一篇文章对kqueue和epoll进行了对比：[Scalable Event Multiplexing: epoll vs. kqueue](http://people.eecs.berkeley.edu/~sangjin/2012/12/21/epoll-vs-kqueue.html)

> The last issue is that epoll does not even support all kinds of file descriptors; select\(\)/poll\(\)/epoll do not work with regular \(disk\) files. This is because epoll has a strong assumption of the readiness model; you monitor the readiness of sockets, so that subsequent IO calls on the sockets do not block. However, disk files do not fit this model, since simply they are always ready.
>
> Disk I/O blocks when the data is not cached in memory, not because the client did not send a message. For disk files, the completion notification model fits. In this model, you simply issue I/O operations on the disk files, and get notified when they are done. kqueue supports this approach with the EVFILT\_AIO filter type, in conjunction with POSIX AIO functions, such as aio\_read\(\). In Linux, you should simply pray that disk access would not block with high cache hit rate \(surprisingly common in many network servers\), or have separate threads so that disk I/O blocking does not affect network socket processing \(e.g., the FLASH architecture\).

这里不再过分展开了，看下这里的代码吧。

```c
#ifndef MILL_FILE_BUFLEN
#define MILL_FILE_BUFLEN (4096)
#endif

// 文件
struct mill_file {
    int fd;
    size_t ifirst;  //file输入缓冲区剩余数据起始位置
    size_t ilen;    //file输入缓冲区剩余数据长度
    size_t olen;    //file输出缓冲区剩余数据长度
    char ibuf[MILL_FILE_BUFLEN];    //file输入缓冲区
    char obuf[MILL_FILE_BUFLEN];    //file输出缓冲区
};

// 文件fd tune操作（设为非阻塞）
static void mill_filetune(int fd) {
    int opt = fcntl(fd, F_GETFL, 0);
    if (opt == -1)
        opt = 0;
    int rc = fcntl(fd, F_SETFL, opt | O_NONBLOCK);
    mill_assert(rc != -1);
}

// 打开文件（fd已调优成非阻塞方式）
struct mill_file *mill_mfopen_(const char *pathname, int flags, mode_t mode) {
    /* Open the file. */
    int fd = open(pathname, flags, mode);
    if (fd == -1)
        return NULL;
    mill_filetune(fd);

    /* Create the object. */
    struct mill_file *f = malloc(sizeof(struct mill_file));
    if(!f) {
        fdclean(fd);
        close(fd);
        errno = ENOMEM;
        return NULL;
    }
    f->fd = fd;
    f->ifirst = 0;
    f->ilen = 0;
    f->olen = 0;
    errno = 0;
    return f;
}

// 文件写操作（这里的写操作对缓冲区的操作逻辑与tcp、unix socket基本一致，不再赘述）
size_t mill_mfwrite_(struct mill_file *f, const void *buf, size_t len, int64_t deadline) {
    /* If it fits into the output buffer copy it there and be done. */
    if(f->olen + len <= MILL_FILE_BUFLEN) {
        memcpy(&f->obuf[f->olen], buf, len);
        f->olen += len;
        errno = 0;
        return len;
    }

    /* If it doesn't fit, flush the output buffer first. */
    mfflush(f, deadline);
    if(errno != 0)
        return 0;

    /* Try to fit it into the buffer once again. */
    if(f->olen + len <= MILL_FILE_BUFLEN) {
        memcpy(&f->obuf[f->olen], buf, len);
        f->olen += len;
        errno = 0;
        return len;
    }

    /* The data chunk to send is longer than the output buffer. Let's do the writing in-place. */
    char *pos = (char*)buf;
    size_t remaining = len;
    while(remaining) {
        ssize_t sz = write(f->fd, pos, remaining);
        if(sz == -1) {
            if(errno != EAGAIN && errno != EWOULDBLOCK)
                return 0;
            int rc = fdwait(f->fd, FDW_OUT, deadline);
            if(rc == 0) {
                errno = ETIMEDOUT;
                return len - remaining;
            }
            mill_assert(rc == FDW_OUT);
            continue;
        }
        pos += sz;
        remaining -= sz;
    }
    return len;
}

// 文件写缓冲flush操作（这里对缓冲区的操作与对tcp、unix socket的操作基本一致，不再赘述）
void mill_mfflush_(struct mill_file *f, int64_t deadline) {
    if(!f->olen) {
        errno = 0;
        return;
    }
    char *pos = f->obuf;
    size_t remaining = f->olen;
    while(remaining) {
        ssize_t sz = write(f->fd, pos, remaining);
        if(sz == -1) {
            if(errno != EAGAIN && errno != EWOULDBLOCK)
                return;
            int rc = fdwait(f->fd, FDW_OUT, deadline);
            if(rc == 0) {
                errno = ETIMEDOUT;
                return;
            }
            mill_assert(rc == FDW_OUT);
            continue;
        }
        pos += sz;
        remaining -= sz;
    }
    f->olen = 0;
    errno = 0;
}

// 文件读操作（这里的读操作对缓冲区的操作逻辑与tcp、unix socket基本一致，不再赘述）
size_t mill_mfread_(struct mill_file *f, void *buf, size_t len, int64_t deadline) {
    /* If there's enough data in the buffer it's easy. */
    if(f->ilen >= len) {
        memcpy(buf, &f->ibuf[f->ifirst], len);
        f->ifirst += len;
        f->ilen -= len;
        errno = 0;
        return len;
    }

    /* Let's move all the data from the buffer first. */
    char *pos = (char*)buf;
    size_t remaining = len;
    memcpy(pos, &f->ibuf[f->ifirst], f->ilen);
    pos += f->ilen;
    remaining -= f->ilen;
    f->ifirst = 0;
    f->ilen = 0;

    mill_assert(remaining);
    while(1) {
        if(remaining > MILL_FILE_BUFLEN) {
            /* If we still have a lot to read try to read it in one go directly
             into the destination buffer. */
            ssize_t sz = read(f->fd, pos, remaining);
            if(!sz) {
                return len - remaining;
            }
            if(sz == -1) {
                if(errno != EAGAIN && errno != EWOULDBLOCK)
                    return len - remaining;
                sz = 0;
            }
            if((size_t)sz == remaining) {
                errno = 0;
                return len;
            }
            pos += sz;
            remaining -= sz;
            if (sz != 0 && mfeof(f)) {
                return len - remaining;
            }
        }
        else {
            /* If we have just a little to read try to read the full connection
             buffer to minimise the number of system calls. */
            ssize_t sz = read(f->fd, f->ibuf, MILL_FILE_BUFLEN);
            if(!sz) {
                return len - remaining;
            }
            if(sz == -1) {
                if(errno != EAGAIN && errno != EWOULDBLOCK)
                    return len - remaining;
                sz = 0;
            }
            if((size_t)sz < remaining) {
                memcpy(pos, f->ibuf, sz);
                pos += sz;
                remaining -= sz;
                f->ifirst = 0;
                f->ilen = 0;
            }
            else {
                memcpy(pos, f->ibuf, remaining);
                f->ifirst = remaining;
                f->ilen = sz - remaining;
                errno = 0;
                return len;
            }
            if (sz != 0 && mfeof(f)) {
                return len - remaining;
            }
        }

        /* Wait till there's more data to read. */
        int res = fdwait(f->fd, FDW_IN, deadline);
        if (!res) {
            errno = ETIMEDOUT;
            return len - remaining;
        }
    }
}

// 文件关闭
void mill_mfclose_(struct mill_file *f) {
    fdclean(f->fd);
    int rc = close(f->fd);
    mill_assert(rc == 0);
    free(f);
    return;
}

// 文件读写位置查看
off_t mill_mftell_(struct mill_file *f) {
    return lseek(f->fd, 0, SEEK_CUR) - f->ilen;
}

// 文件读写位置定位
off_t mill_mfseek_(struct mill_file *f, off_t offset) {
    f->ifirst = 0;
    f->ilen = 0;
    f->olen = 0;
    return lseek(f->fd, offset, SEEK_SET);
}

// 判断是否读到文件末尾
int mill_mfeof_(struct mill_file *f) {
    // 首先获取当前位置
    off_t current = lseek(f->fd, 0, SEEK_CUR);
    if (current == -1)
        return -1;
    // 再获取文件末尾位置
    off_t eof = lseek(f->fd, 0, SEEK_END);
    if (eof == -1)
        return -1;
    // 恢复读写位置为之前的读写位置
    off_t res = lseek(f->fd, current, SEEK_SET);
    if (res == -1)
        return -1;
    // 比较是否到达文件末尾位置
    return (current == eof);
}

// stdin
struct mill_file *mill_mfin_(void) {
    static struct mill_file f = {-1, 0, 0, 0};
    if(mill_slow(f.fd < 0)) {
        mill_filetune(STDIN_FILENO);
        f.fd = STDIN_FILENO;
    }
    return &f;
}

// stdout
struct mill_file *mill_mfout_(void) {
    static struct mill_file f = {-1, 0, 0, 0};
    if(mill_slow(f.fd < 0)) {
        mill_filetune(STDOUT_FILENO);
        f.fd = STDOUT_FILENO;
    }
    return &f;
}

// stderr
struct mill_file *mill_mferr_(void) {
    static struct mill_file f = {-1, 0, 0, 0};
    if(mill_slow(f.fd < 0)) {
        mill_filetune(STDERR_FILENO);
        f.fd = STDERR_FILENO;
    }
    return &f;
}
```

