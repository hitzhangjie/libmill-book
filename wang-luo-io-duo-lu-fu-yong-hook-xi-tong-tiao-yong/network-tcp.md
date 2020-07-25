---
description: >-
  libmill是一个基于c语言开发的go风格协程库，实现了go风格的协程操作、chan操作、非阻塞的网络io操作等等，是一个不错的linux平台下c风格协程库实现。如果你想从0到1的快速了解如何开发一个协程库，libmill将是一个非常不错的案例；如果你想更深入地了解go，libmill里面也借鉴了go的一些设计思想；或者你想在生产环境中使用，开发者也提供了一个更健壮的版本libdill。
---

# network: tcp



#### tcp

```c
/* The buffer size is based on typical Ethernet MTU (1500 bytes). Making it
   smaller would yield small suboptimal packets. Making it higher would bring
   no substantial benefit. The value is made smaller to account for IPv4/IPv6
   and TCP headers. Few more bytes are subtracted to account for any possible
   IP or TCP options */

// tcp接收缓冲大小
// 根据Ethernet MTU（1500字节）进行配置的，小了不是最优但是大了也没有实质好处
// 这里设置的小一点以满足IPv4/IPv6的headers，多省几个字节出来可以控制IP、TCP选项
#ifndef MILL_TCP_BUFLEN
#define MILL_TCP_BUFLEN (1500 - 68)
#endif

// tcp socket 类型
enum mill_tcptype {
   MILL_TCPLISTENER,
   MILL_TCPCONN
};

struct mill_tcpsock_ {
    enum mill_tcptype type;
};

// tcp listen socket
struct mill_tcplistener {
    struct mill_tcpsock_ sock;
    int fd;
    int port;
};

// tcp conn socket
struct mill_tcpconn {
    struct mill_tcpsock_ sock;
    int fd;
    size_t ifirst;  // 接收缓冲区剩余数据的起始位置
    size_t ilen;    // 接收缓冲区剩余数据的长度
    size_t olen;    // 发送缓冲区剩余数据的长度
    char ibuf[MILL_TCP_BUFLEN]; // 接收缓冲区
    char obuf[MILL_TCP_BUFLEN]; // 发送缓冲区
    ipaddr addr;    // peer socket地址
};

// tcp socket tune（设为非阻塞模式、地址立即可重用(主动关闭不进入timed wait）、屏蔽SIGPIPE）
static void mill_tcptune(int s) {
    /* Make the socket non-blocking. */
    int opt = fcntl(s, F_GETFL, 0);
    if (opt == -1)
        opt = 0;
    int rc = fcntl(s, F_SETFL, opt | O_NONBLOCK);
    mill_assert(rc != -1);
    /*  Allow re-using the same local address rapidly. */
    opt = 1;
    rc = setsockopt(s, SOL_SOCKET, SO_REUSEADDR, &opt, sizeof (opt));
    mill_assert(rc == 0);
    /* If possible, prevent SIGPIPE signal when writing to the connection
        already closed by the peer. */
#ifdef SO_NOSIGPIPE
    opt = 1;
    rc = setsockopt (s, SOL_SOCKET, SO_NOSIGPIPE, &opt, sizeof (opt));
    mill_assert (rc == 0 || errno == EINVAL);
#endif
}

// tcp conn socket 初始化
static void tcpconn_init(struct mill_tcpconn *conn, int fd) {
    conn->sock.type = MILL_TCPCONN;
    conn->fd = fd;
    conn->ifirst = 0;
    conn->ilen = 0;
    conn->olen = 0;
}

// tcp listen socket 初始化（非阻塞socket）
struct mill_tcpsock_ *mill_tcplisten_(ipaddr addr, int backlog) {
    /* Open the listening socket. */
    int s = socket(mill_ipfamily(addr), SOCK_STREAM, 0);
    if(s == -1)
        return NULL;
    mill_tcptune(s);

    /* Start listening. */
    int rc = bind(s, (struct sockaddr*)&addr, mill_iplen(addr));
    if(rc != 0)
        return NULL;
    rc = listen(s, backlog);
    if(rc != 0)
        return NULL;

    // 如果参数addr中没有指定port信息，bind的时候os回自动分配一个，这里需要获取分配的port
    int port = mill_ipport(addr);
    if(!port) {
        ipaddr baddr;
        socklen_t len = sizeof(ipaddr);
        rc = getsockname(s, (struct sockaddr*)&baddr, &len);
        if(rc == -1) {
            int err = errno;
            fdclean(s);
            close(s);
            errno = err;
            return NULL;
        }
        port = mill_ipport(baddr);
    }

    /* Create the object. */
    struct mill_tcplistener *l = malloc(sizeof(struct mill_tcplistener));
    if(!l) {
        fdclean(s);
        close(s);
        errno = ENOMEM;
        return NULL;
    }
    l->sock.type = MILL_TCPLISTENER;
    l->fd = s;
    l->port = port;
    errno = 0;
    return &l->sock;
}

// 获取peer socket对应的port信息（统一处理连接socket和监听socket）
int mill_tcpport_(struct mill_tcpsock_ *s) {
    if(s->type == MILL_TCPCONN) {
        struct mill_tcpconn *c = (struct mill_tcpconn*)s;
        return mill_ipport(c->addr);
    }
    else if(s->type == MILL_TCPLISTENER) {
        struct mill_tcplistener *l = (struct mill_tcplistener*)s;
        return l->port;
    }
    mill_assert(0);
}

// tcp listen socket 接受一个连接（非阻塞方式accept，tcp conn socket设为非阻塞）
struct mill_tcpsock_ *mill_tcpaccept_(struct mill_tcpsock_ *s, int64_t deadline) {
    if(s->type != MILL_TCPLISTENER)
        mill_panic("trying to accept on a socket that isn't listening");
    struct mill_tcplistener *l = (struct mill_tcplistener*)s;
    socklen_t addrlen;
    ipaddr addr;
    while(1) {
        /* Try to get new connection (non-blocking). */
        addrlen = sizeof(addr);
        int as = accept(l->fd, (struct sockaddr *)&addr, &addrlen);
        if (as >= 0) {
            mill_tcptune(as);
            struct mill_tcpconn *conn = malloc(sizeof(struct mill_tcpconn));
            if(!conn) {
                fdclean(as);
                close(as);
                errno = ENOMEM;
                return NULL;
            }
            tcpconn_init(conn, as);
            conn->addr = addr;
            errno = 0;
            return (tcpsock)conn;
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

// tcp conn socket 连接到指定地址（非阻塞方式）
struct mill_tcpsock_ *mill_tcpconnect_(ipaddr addr, int64_t deadline) {
    /* Open a socket. */
    int s = socket(mill_ipfamily(addr), SOCK_STREAM, 0);
    if(s == -1)
        return NULL;
    mill_tcptune(s);

    /* Connect to the remote endpoint. */
    int rc = connect(s, (struct sockaddr*)&addr, mill_iplen(addr));
    if(rc != 0) {
        mill_assert(rc == -1);
        if(errno != EINPROGRESS)
            return NULL;
        rc = fdwait(s, FDW_OUT, deadline);
        if(rc == 0) {
            errno = ETIMEDOUT;
            return NULL;
        }
        int err;
        socklen_t errsz = sizeof(err);
        rc = getsockopt(s, SOL_SOCKET, SO_ERROR, (void*)&err, &errsz);
        if(rc != 0) {
            err = errno;
            fdclean(s);
            close(s);
            errno = err;
            return NULL;
        }
        if(err != 0) {
            fdclean(s);
            close(s);
            errno = err;
            return NULL;
        }
    }

    /* Create the object. */
    struct mill_tcpconn *conn = malloc(sizeof(struct mill_tcpconn));
    if(!conn) {
        fdclean(s);
        close(s);
        errno = ENOMEM;
        return NULL;
    }
    tcpconn_init(conn, s);
    errno = 0;
    return (tcpsock)conn;
}

// tcp socket 发送（非阻塞方式）
size_t mill_tcpsend_(struct mill_tcpsock_ *s, const void *buf, size_t len, int64_t deadline) {
    if(s->type != MILL_TCPCONN)
        mill_panic("trying to send to an unconnected socket");
    struct mill_tcpconn *conn = (struct mill_tcpconn*)s;

    // 如果发送缓冲区剩余空间可以容纳待发送数据，直接拷贝到发送缓冲
    if(conn->olen + len <= MILL_TCP_BUFLEN) {
        memcpy(&conn->obuf[conn->olen], buf, len);
        conn->olen += len;
        errno = 0;
        return len;
    }

    // 如果剩余空间不太够，则先发送完发送缓冲中的数据
    tcpflush(s, deadline);
    if(errno != 0)
        return 0;

    // tcpflush不一定全部发送完（如超时返回）
    // 继续检查剩余空间是不是够容纳带发送缓冲，可以则直接拷贝到发送缓冲
    if(conn->olen + len <= MILL_TCP_BUFLEN) {
        memcpy(&conn->obuf[conn->olen], buf, len);
        conn->olen += len;
        errno = 0;
        return len;
    }

    // 尝试tcpflush之后发送缓冲还是不够大则直接就地发送，即以指定buf为发送缓冲进行发送
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

// tcp conn socket flush发送缓冲数据（非阻塞方式）
void mill_tcpflush_(struct mill_tcpsock_ *s, int64_t deadline) {
    if(s->type != MILL_TCPCONN)
        mill_panic("trying to send to an unconnected socket");
    struct mill_tcpconn *conn = (struct mill_tcpconn*)s;
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

// tcp conn socket 接收数据（非阻塞方式）
size_t mill_tcprecv_(struct mill_tcpsock_ *s, void *buf, size_t len, int64_t deadline) {
    if(s->type != MILL_TCPCONN)
        mill_panic("trying to receive from an unconnected socket");
    struct mill_tcpconn *conn = (struct mill_tcpconn*)s;

    // 如果接收缓冲中有足够多数据，则直接返回len长度的数据给buf
    if(conn->ilen >= len) {
        memcpy(buf, &conn->ibuf[conn->ifirst], len);
        conn->ifirst += len;
        conn->ilen -= len;
        errno = 0;
        return len;
    }

    // 接收缓冲中数据少于请求数据量，先拷贝已接收的部分，再继续接收数据
    char *pos = (char*)buf;
    size_t remaining = len;
    memcpy(pos, &conn->ibuf[conn->ifirst], conn->ilen);
    pos += conn->ilen;
    remaining -= conn->ilen;
    conn->ifirst = 0;
    conn->ilen = 0;

    // 继续接收剩余数据
    mill_assert(remaining);
    while(1) {

        if(remaining > MILL_TCP_BUFLEN) {
            // 如果请求数据量大于tcp接收缓冲大小，为了减少系统调用次数直接就地recv
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
            // 剩余请求数据量小于接收缓存大小，但是仍按MILL_TCP_BUFLEN进行接收，
            // 这样可以减少后续mill_tcprecv时调用系统调用recv的次数
            ssize_t sz = recv(conn->fd, conn->ibuf, MILL_TCP_BUFLEN, 0);
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

        // 等待数据可读事件就绪，继续读取更多数据
        int res = fdwait(conn->fd, FDW_IN, deadline);
        if(!res) {
            errno = ETIMEDOUT;
            return len - remaining;
        }
    }
}

// tcp conn socket 接收数据（遇到指定分界符会停止接收数据，非阻塞工作方式）
size_t mill_tcprecvuntil_(struct mill_tcpsock_ *s, void *buf, size_t len,
      const char *delims, size_t delimcount, int64_t deadline) {
    if(s->type != MILL_TCPCONN)
        mill_panic("trying to receive from an unconnected socket");
    char *pos = (char*)buf;
    size_t i;
    for(i = 0; i != len; ++i, ++pos) {
        size_t res = tcprecv(s, pos, 1, deadline);
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

// tcp conn socket 关闭（读关闭或写关闭或both）
void mill_tcpshutdown_(struct mill_tcpsock_ *s, int how) {
    mill_assert(s->type == MILL_TCPCONN);
    struct mill_tcpconn *c = (struct mill_tcpconn*)s;
    int rc = shutdown(c->fd, how);
    mill_assert(rc == 0 || errno == ENOTCONN);
}

// tcp socket 关闭（统一处理listen socket和conn socket）
void mill_tcpclose_(struct mill_tcpsock_ *s) {
    if(s->type == MILL_TCPLISTENER) {
        struct mill_tcplistener *l = (struct mill_tcplistener*)s;
        fdclean(l->fd);
        int rc = close(l->fd);
        mill_assert(rc == 0);
        free(l);
        return;
    }
    if(s->type == MILL_TCPCONN) {
        struct mill_tcpconn *c = (struct mill_tcpconn*)s;
        fdclean(c->fd);
        int rc = close(c->fd);
        mill_assert(rc == 0);
        free(c);
        return;
    }
    mill_assert(0);
}

// 获取tcp连接peer socket地址
ipaddr mill_tcpaddr_(struct mill_tcpsock_ *s) {
    if(s->type != MILL_TCPCONN)
        mill_panic("trying to get address from a socket that isn't connected");
    struct mill_tcpconn *l = (struct mill_tcpconn *)s;
    return l->addr;
}

/* This function is to be used only internally by libmill. Take into account
   that once there are data in tcpsock's tx/rx buffers, the state of fd may
   not match the state of tcpsock object. Works only on connected sockets. */
int mill_tcpfd(struct mill_tcpsock_ *s) {
    return ((struct mill_tcpconn*)s)->fd;
}
```

