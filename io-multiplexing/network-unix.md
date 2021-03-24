---
description: >-
  libmill是一个基于c语言开发的go风格协程库，实现了go风格的协程操作、chan操作、非阻塞的网络io操作等等，是一个不错的linux平台下c风格协程库实现。如果你想从0到1的快速了解如何开发一个协程库，libmill将是一个非常不错的案例；如果你想更深入地了解go，libmill里面也借鉴了go的一些设计思想；或者你想在生产环境中使用，开发者也提供了一个更健壮的版本libdill。
---

# network: unix



#### unix-socket

```c
// unix socket接收缓冲大小
#ifndef MILL_UNIX_BUFLEN
#define MILL_UNIX_BUFLEN (4096)
#endif

// unix socket类型
enum mill_unixtype {
   MILL_UNIXLISTENER,
   MILL_UNIXCONN
};

// unix socket
struct mill_unixsock_ {
    enum mill_unixtype type;
};

// unix listen socket
struct mill_unixlistener {
    struct mill_unixsock_ sock;
    int fd;
};

// unix conn socket
struct mill_unixconn {
    struct mill_unixsock_ sock;
    int fd;
    size_t ifirst;  // 接收缓冲区中剩余数据起始位置
    size_t ilen;    // 接收缓冲区中剩余数据长度
    size_t olen;    // 发送缓冲区中剩余数据长度
    char ibuf[MILL_UNIX_BUFLEN];    // 接收缓冲区
    char obuf[MILL_UNIX_BUFLEN];    // 发送缓冲区
};

// unix socket tune (设为nonblocking & 屏蔽SIGPIPE)
static void mill_unixtune(int s) {
    /* Make the socket non-blocking. */
    int opt = fcntl(s, F_GETFL, 0);
    if (opt == -1)
        opt = 0;
    int rc = fcntl(s, F_SETFL, opt | O_NONBLOCK);
    mill_assert(rc != -1);
    /* If possible, prevent SIGPIPE signal when writing to the connection
        already closed by the peer. */
#ifdef SO_NOSIGPIPE
    opt = 1;
    rc = setsockopt (s, SOL_SOCKET, SO_NOSIGPIPE, &opt, sizeof (opt));
    mill_assert (rc == 0 || errno == EINVAL);
#endif
}

// unix socket 地址设置
static int mill_unixresolve(const char *addr, struct sockaddr_un *su) {
    mill_assert(su);
    if (strlen(addr) >= sizeof(su->sun_path)) {
        errno = EINVAL;
        return -1;
    }
    su->sun_family = AF_UNIX;
    strncpy(su->sun_path, addr, sizeof(su->sun_path));
    errno = 0;
    return 0;
}

// unix conn socket 初始化
static void unixconn_init(struct mill_unixconn *conn, int fd) {
    conn->sock.type = MILL_UNIXCONN;
    conn->fd = fd;
    conn->ifirst = 0;
    conn->ilen = 0;
    conn->olen = 0;
}

// unix listen socket 初始化
struct mill_unixsock_ *mill_unixlisten_(const char *addr, int backlog) {
    struct sockaddr_un su;
    int rc = mill_unixresolve(addr, &su);
    if (rc != 0) {
        return NULL;
    }
    /* Open the listening socket. */
    int s = socket(AF_UNIX, SOCK_STREAM, 0);
    if(s == -1)
        return NULL;
    mill_unixtune(s);

    /* Start listening. */
    rc = bind(s, (struct sockaddr*)&su, sizeof(struct sockaddr_un));
    if(rc != 0)
        return NULL;
    rc = listen(s, backlog);
    if(rc != 0)
        return NULL;

    /* Create the object. */
    struct mill_unixlistener *l = malloc(sizeof(struct mill_unixlistener));
    if(!l) {
        fdclean(s);
        close(s);
        errno = ENOMEM;
        return NULL;
    }
    l->sock.type = MILL_UNIXLISTENER;
    l->fd = s;
    errno = 0;
    return &l->sock;
}

// unix listen socket 接受一个连接请求（非阻塞方式）
struct mill_unixsock_ *mill_unixaccept_(struct mill_unixsock_ *s, int64_t deadline) {
    if(s->type != MILL_UNIXLISTENER)
        mill_panic("trying to accept on a socket that isn't listening");
    struct mill_unixlistener *l = (struct mill_unixlistener*)s;
    while(1) {
        /* Try to get new connection (non-blocking). */
        int as = accept(l->fd, NULL, NULL);
        if (as >= 0) {
            mill_unixtune(as);
            struct mill_unixconn *conn = malloc(sizeof(struct mill_unixconn));
            if(!conn) {
                fdclean(as);
                close(as);
                errno = ENOMEM;
                return NULL;
            }
            unixconn_init(conn, as);
            errno = 0;
            return (struct mill_unixsock_*)conn;
        }
        mill_assert(as == -1);
        if(errno != EAGAIN && errno != EWOULDBLOCK)
            return NULL;
        /* Wait till new connection is available. */
        int rc = fdwait(l->fd, FDW_IN, deadline);
        if(rc == 0) {
            errno = ETIMEDOUT;
            return NULL;
        }
        if(rc & FDW_ERR)
            return NULL;
        mill_assert(rc == FDW_IN);
    }
}

// unix socket 连接到服务地址（非阻塞方式）
struct mill_unixsock_ *mill_unixconnect_(const char *addr) {
    struct sockaddr_un su;
    int rc = mill_unixresolve(addr, &su);
    if (rc != 0) {
        return NULL;
    }

    /* Open a socket. */
    int s = socket(AF_UNIX,  SOCK_STREAM, 0);
    if(s == -1)
        return NULL;
    mill_unixtune(s);

    /* Connect to the remote endpoint. */
    rc = connect(s, (struct sockaddr*)&su, sizeof(struct sockaddr_un));
    if(rc != 0) {
        int err = errno;
        mill_assert(rc == -1);
        fdclean(s);
        close(s);
        errno = err;
        return NULL;
    }

    /* Create the object. */
    struct mill_unixconn *conn = malloc(sizeof(struct mill_unixconn));
    if(!conn) {
        fdclean(s);
        close(s);
        errno = ENOMEM;
        return NULL;
    }
    unixconn_init(conn, s);
    errno = 0;
    return (struct mill_unixsock_*)conn;
}

// 创建 unix socket pair （非阻塞方式）
void mill_unixpair_(struct mill_unixsock_ **a, struct mill_unixsock_ **b) {
    if(!a || !b) {
        errno = EINVAL;
        return;
    }
    int fd[2];
    int rc = socketpair(AF_UNIX, SOCK_STREAM, 0, fd);
    if (rc != 0)
        return;
    mill_unixtune(fd[0]);
    mill_unixtune(fd[1]);
    struct mill_unixconn *conn = malloc(sizeof(struct mill_unixconn));
    if(!conn) {
        fdclean(fd[0]);
        close(fd[0]);
        fdclean(fd[1]);
        close(fd[1]);
        errno = ENOMEM;
        return;
    }
    unixconn_init(conn, fd[0]);
    *a = (struct mill_unixsock_*)conn;
    conn = malloc(sizeof(struct mill_unixconn));
    if(!conn) {
        free(*a);
        fdclean(fd[0]);
        close(fd[0]);
        fdclean(fd[1]);
        close(fd[1]);
        errno = ENOMEM;
        return;
    }
    unixconn_init(conn, fd[1]);
    *b = (struct mill_unixsock_*)conn;
    errno = 0;
}

// unix socket send（非阻塞方式）
size_t mill_unixsend_(struct mill_unixsock_ *s, const void *buf, size_t len, int64_t deadline) {
    if(s->type != MILL_UNIXCONN)
        mill_panic("trying to send to an unconnected socket");
    struct mill_unixconn *conn = (struct mill_unixconn*)s;

    // 如果输出缓冲中剩余空间可以容纳待发送数据，则将待发送数据直接拷贝到输出缓冲
    if(conn->olen + len <= MILL_UNIX_BUFLEN) {
        memcpy(&conn->obuf[conn->olen], buf, len);
        conn->olen += len;
        errno = 0;
        return len;
    }

    // 如果输出缓冲剩余空间不能容纳待发送数据，则先发送输出缓冲中数据腾空间
    unixflush(s, deadline);
    if(errno != 0)
        return 0;

    // unixflush不一定把数据发送完（超时返回情况下），需再次检查剩余空间是否能容纳待发送数据，
    // 能容纳则直接将待发送数据拷贝到输出缓冲区中
    if(conn->olen + len <= MILL_UNIX_BUFLEN) {
        memcpy(&conn->obuf[conn->olen], buf, len);
        conn->olen += len;
        errno = 0;
        return len;
    }

    // 经过上述unixflush处理后，发送缓冲区还是无法容纳下待发送数据，则直接就地发送待发送数据
    char *pos = (char*)buf;
    size_t remaining = len;
    while(remaining) {
        ssize_t sz = send(conn->fd, pos, remaining, 0);
        if(sz == -1) {
            /* Operating systems are inconsistent w.r.t. returning EPIPE and
               ECONNRESET. Let's paper over it like this. */
            if(errno == EPIPE) {
                errno = ECONNRESET;
                return 0;
            }
            if(errno != EAGAIN && errno != EWOULDBLOCK)
                return 0;
            int rc = fdwait(conn->fd, FDW_OUT, deadline);
            if(rc == 0) {
                errno = ETIMEDOUT;
                return len - remaining;
            }
            continue;
        }
        pos += sz;
        remaining -= sz;
    }
    errno = 0;
    return len;
}

// unix socket 将发送缓冲区中的数据发送出去（非阻塞方式，超时返回停止发送）
void mill_unixflush_(struct mill_unixsock_ *s, int64_t deadline) {
    if(s->type != MILL_UNIXCONN)
        mill_panic("trying to send to an unconnected socket");
    struct mill_unixconn *conn = (struct mill_unixconn*)s;
    if(!conn->olen) {
        errno = 0;
        return;
    }
    char *pos = conn->obuf;
    size_t remaining = conn->olen;
    while(remaining) {
        ssize_t sz = send(conn->fd, pos, remaining, 0);
        if(sz == -1) {
            /* Operating systems are inconsistent w.r.t. returning EPIPE and
               ECONNRESET. Let's paper over it like this. */
            if(errno == EPIPE) {
                errno = ECONNRESET;
                return;
            }
            if(errno != EAGAIN && errno != EWOULDBLOCK)
                return;
            int rc = fdwait(conn->fd, FDW_OUT, deadline);
            if(rc == 0) {
                errno = ETIMEDOUT;
                return;
            }
            continue;
        }
        pos += sz;
        remaining -= sz;
    }
    conn->olen = 0;
    errno = 0;
}

// unix socket recv（非阻塞方式）
size_t mill_unixrecv_(struct mill_unixsock_ *s, void *buf, size_t len, int64_t deadline) {
    if(s->type != MILL_UNIXCONN)
        mill_panic("trying to receive from an unconnected socket");
    struct mill_unixconn *conn = (struct mill_unixconn*)s;

    // 如果接收缓冲区中有足够的数据，直接拷贝到用户指定buf即可
    if(conn->ilen >= len) {
        memcpy(buf, &conn->ibuf[conn->ifirst], len);
        conn->ifirst += len;
        conn->ilen -= len;
        errno = 0;
        return len;
    }

    // 如果接收缓冲区中没有足够的数据，则先拷贝有的数据到用户指定buf，再继续接收剩余数据
    char *pos = (char*)buf;
    size_t remaining = len;
    memcpy(pos, &conn->ibuf[conn->ifirst], conn->ilen);
    pos += conn->ilen;
    remaining -= conn->ilen;
    conn->ifirst = 0;
    conn->ilen = 0;

    // 继续读取剩余数据
    mill_assert(remaining);
    while(1) {
        if(remaining > MILL_UNIX_BUFLEN) {
            // 如果还有很多数据要读，为减少系统调用次数直接将数据读取到用户指定buf
            ssize_t sz = recv(conn->fd, pos, remaining, 0);
            if(!sz) {
                errno = ECONNRESET;
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
        }
        else {
            // 如果要接收的剩余数据不多了，则接收到接收缓冲区中，再拷贝到用户指定buf
            // 接收MILL_UNIX_BUFLEN的数据，目的是减少之后mill_recv可能引发的recv系统调用的次数
            ssize_t sz = recv(conn->fd, conn->ibuf, MILL_UNIX_BUFLEN, 0);
            if(!sz) {
                errno = ECONNRESET;
                return len - remaining;
            }
            if(sz == -1) {
                if(errno != EAGAIN && errno != EWOULDBLOCK)
                    return len - remaining;
                sz = 0;
            }
            if((size_t)sz < remaining) {
                memcpy(pos, conn->ibuf, sz);
                pos += sz;
                remaining -= sz;
                conn->ifirst = 0;
                conn->ilen = 0;
            }
            else {
                memcpy(pos, conn->ibuf, remaining);
                conn->ifirst = remaining;
                conn->ilen = sz - remaining;
                errno = 0;
                return len;
            }
        }

        // 继续等待后续数据可读事件到达，然后读取（若超时则返回停止接收）
        int res = fdwait(conn->fd, FDW_IN, deadline);
        if(!res) {
            errno = ETIMEDOUT;
            return len - remaining;
        }
    }
}

// unix socket recv 读取len bytes到buf，读取到指定delimiter会停止读取（非阻塞方式）
size_t mill_unixrecvuntil_(struct mill_unixsock_ *s, void *buf, size_t len,
      const char *delims, size_t delimcount, int64_t deadline) {
    if(s->type != MILL_UNIXCONN)
        mill_panic("trying to receive from an unconnected socket");
    unsigned char *pos = (unsigned char*)buf;
    size_t i;
    for(i = 0; i != len; ++i, ++pos) {
        size_t res = unixrecv(s, pos, 1, deadline);
        if(res == 1) {
            size_t j;
            for(j = 0; j != delimcount; ++j)
                if(*pos == delims[j])
                    return i + 1;
        }
        if (errno != 0)
            return i + res;
    }
    errno = ENOBUFS;
    return len;
}

// 关闭socket，写关闭或者读关闭或者both
void mill_unixshutdown_(struct mill_unixsock_ *s, int how) {
    mill_assert(s->type == MILL_UNIXCONN);
    struct mill_unixconn *c = (struct mill_unixconn*)s;
    int rc = shutdown(c->fd, how);
    mill_assert(rc == 0 || errno == ENOTCONN);
}

// 关闭unix socket（统一处理监听socket和连接socket）
void mill_unixclose_(struct mill_unixsock_ *s) {
    if(s->type == MILL_UNIXLISTENER) {
        struct mill_unixlistener *l = (struct mill_unixlistener*)s;
        fdclean(l->fd);
        int rc = close(l->fd);
        mill_assert(rc == 0);
        free(l);
        return;
    }
    if(s->type == MILL_UNIXCONN) {
        struct mill_unixconn *c = (struct mill_unixconn*)s;
        fdclean(c->fd);
        int rc = close(c->fd);
        mill_assert(rc == 0);
        free(c);
        return;
    }
    mill_assert(0);
}
```

