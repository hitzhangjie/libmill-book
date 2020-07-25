---
description: >-
  libmill是一个基于c语言开发的go风格协程库，实现了go风格的协程操作、chan操作、非阻塞的网络io操作等等，是一个不错的linux平台下c风格协程库实现。如果你想从0到1的快速了解如何开发一个协程库，libmill将是一个非常不错的案例；如果你想更深入地了解go，libmill里面也借鉴了go的一些设计思想；或者你想在生产环境中使用，开发者也提供了一个更健壮的版本libdill。
---

# network: udp

#### udp

```c
// udp socket
struct mill_udpsock_ {
    int fd;
    int port;
};

// udp socket tune（设置为nonblocking）
static void mill_udptune(int s) {
    /* Make the socket non-blocking. */
    int opt = fcntl(s, F_GETFL, 0);
    if (opt == -1)
        opt = 0;
    int rc = fcntl(s, F_SETFL, opt | O_NONBLOCK);
    mill_assert(rc != -1);
}

// udp listen socket 创建（socket为非阻塞）
struct mill_udpsock_ *mill_udplisten_(ipaddr addr) {
    /* Open the listening socket. */
    int s = socket(mill_ipfamily(addr), SOCK_DGRAM, 0);
    if(s == -1)
        return NULL;
    mill_udptune(s);

    /* Start listening. */
    int rc = bind(s, (struct sockaddr*)&addr, mill_iplen(addr));
    if(rc != 0)
        return NULL;

    // 参数addr中可能没有指定port信息（用户可能希望监听指定ip、自动分配的port），
    // 此时需要获取os自动分配的port信息
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
    struct mill_udpsock_ *us = malloc(sizeof(struct mill_udpsock_));
    if(!us) {
        fdclean(s);
        close(s);
        errno = ENOMEM;
        return NULL;
    }
    us->fd = s;
    us->port = port;
    errno = 0;
    return us;
}

// unix socket 获取绑定的port
int mill_udpport_(struct mill_udpsock_ *s) {
    return s->port;
}

// udp socket 发送（socket需提前设置为非阻塞方式）
void mill_udpsend_(struct mill_udpsock_ *s, ipaddr addr, const void *buf, size_t len) {
    struct sockaddr *saddr = (struct sockaddr*) &addr;
    ssize_t ss = sendto(s->fd, buf, len, 0, saddr, saddr->sa_family ==
        AF_INET ? sizeof(struct sockaddr_in) : sizeof(struct sockaddr_in6));
    if(mill_fast(ss == (ssize_t)len)) {
        errno = 0;
        return;
    }
    mill_assert(ss < 0);
    if(errno == EAGAIN || errno == EWOULDBLOCK)
        errno = 0;
}

// udp socket 接收（socket需提前设置为非阻塞方式）
size_t mill_udprecv_(struct mill_udpsock_ *s, ipaddr *addr, void *buf, size_t len, int64_t deadline) {
    ssize_t ss;
    while(1) {
        socklen_t slen = sizeof(ipaddr);
        ss = recvfrom(s->fd, buf, len, 0, (struct sockaddr*)addr, &slen);
        if(ss >= 0)
            break;
        if(errno != EAGAIN && errno != EWOULDBLOCK)
            return 0;
        int rc = fdwait(s->fd, FDW_IN, deadline);
        if(rc == 0) {
            errno = ETIMEDOUT;
            return 0;
        }
    }
    errno = 0;
    return (size_t)ss;
}

// udp socket 关闭
void mill_udpclose_(struct mill_udpsock_ *s) {
    fdclean(s->fd);
    int rc = close(s->fd);
    mill_assert(rc == 0);
    free(s);
}
```

这里udp socket接收、发送之前需要自己显示地进行mill\_udptune\(int s\)，这个感觉封装地不友好。

