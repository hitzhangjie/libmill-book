---
description: >-
  libmill是一个基于c语言开发的go风格协程库，实现了go风格的协程操作、chan操作、非阻塞的网络io操作等等，是一个不错的linux平台下c风格协程库实现。如果你想从0到1的快速了解如何开发一个协程库，libmill将是一个非常不错的案例；如果你想更深入地了解go，libmill里面也借鉴了go的一些设计思想；或者你想在生产环境中使用，开发者也提供了一个更健壮的版本libdill。
---

# network: ip

**file: ip.h**

```c
int mill_ipfamily(ipaddr addr);
int mill_iplen(ipaddr addr);
int mill_ipport(ipaddr addr);
```

**file: ip.c**

```c
MILL_CT_ASSERT(sizeof(ipaddr) >= sizeof(struct sockaddr_in));
MILL_CT_ASSERT(sizeof(ipaddr) >= sizeof(struct sockaddr_in6));

static struct dns_resolv_conf *mill_dns_conf = NULL;
static struct dns_hosts *mill_dns_hosts = NULL;
static struct dns_hints *mill_dns_hints = NULL;

// 创建一个ip地址为INADDR_ANY、指定端口port、mode=ipv4/ipv6的ip地址
static ipaddr mill_ipany(int port, int mode)
{
    ipaddr addr;
    if(mill_slow(port < 0 || port > 0xffff)) {
        ((struct sockaddr*)&addr)->sa_family = AF_UNSPEC;
        errno = EINVAL;
        return addr;
    }
    if (mode == 0 || mode == IPADDR_IPV4 || mode == IPADDR_PREF_IPV4) {
        struct sockaddr_in *ipv4 = (struct sockaddr_in*)&addr;
        ipv4->sin_family = AF_INET;
        ipv4->sin_addr.s_addr = htonl(INADDR_ANY);
        ipv4->sin_port = htons((uint16_t)port);
    }
    else {
        struct sockaddr_in6 *ipv6 = (struct sockaddr_in6*)&addr;
        ipv6->sin6_family = AF_INET6;
        memcpy(&ipv6->sin6_addr, &in6addr_any, sizeof(in6addr_any));
        ipv6->sin6_port = htons((uint16_t)port);
    }
    errno = 0;
    return addr;
}

// 将点分十进制ipv4地址:端口号=ip:port转换为二进制格式（已转为字节序）
static ipaddr mill_ipv4_literal(const char *addr, int port) {
    ipaddr raddr;
    struct sockaddr_in *ipv4 = (struct sockaddr_in*)&raddr;
    int rc = inet_pton(AF_INET, addr, &ipv4->sin_addr);
    mill_assert(rc >= 0);
    if(rc == 1) {
        ipv4->sin_family = AF_INET;
        ipv4->sin_port = htons((uint16_t)port);
        errno = 0;
        return raddr;
    }
    ipv4->sin_family = AF_UNSPEC;
    errno = EINVAL;
    return raddr;
}

// 将点分ipv4地址:端口号=ipv6:port转换为二进制格式（已转网络字节序）
static ipaddr mill_ipv6_literal(const char *addr, int port) {
    ipaddr raddr;
    struct sockaddr_in6 *ipv6 = (struct sockaddr_in6*)&raddr;
    int rc = inet_pton(AF_INET6, addr, &ipv6->sin6_addr);
    mill_assert(rc >= 0);
    if(rc == 1) {
        ipv6->sin6_family = AF_INET6;
        ipv6->sin6_port = htons((uint16_t)port);
        errno = 0;
        return raddr;
    }
    ipv6->sin6_family = AF_UNSPEC;
    errno = EINVAL;
    return raddr;
}

// 转换ipv4地址或者ipv6地址为二进制地址格式（组合调用上面的函数mill_ipv4/6_literal）
static ipaddr mill_ipliteral(const char *addr, int port, int mode) {
    ipaddr raddr;
    struct sockaddr *sa = (struct sockaddr*)&raddr;
    if(mill_slow(!addr || port < 0 || port > 0xffff)) {
        sa->sa_family = AF_UNSPEC;
        errno = EINVAL;
        return raddr;
    }
    switch(mode) {
        case IPADDR_IPV4:
            return mill_ipv4_literal(addr, port);
        case IPADDR_IPV6:
            return mill_ipv6_literal(addr, port);
        case 0:
        case IPADDR_PREF_IPV4:
            raddr = mill_ipv4_literal(addr, port);
            if(errno == 0)
                return raddr;
            return mill_ipv6_literal(addr, port);
        case IPADDR_PREF_IPV6:
            raddr = mill_ipv6_literal(addr, port);
            if(errno == 0)
                return raddr;
            return mill_ipv4_literal(addr, port);
        default:
            mill_assert(0);
    }
}

// 返回ip地址的地址族
int mill_ipfamily(ipaddr addr) {
    return ((struct sockaddr*)&addr)->sa_family;
}

// 返回ip地址结构体长度
int mill_iplen(ipaddr addr) {
    return mill_ipfamily(addr) == AF_INET ? 
                        sizeof(struct sockaddr_in) : sizeof(struct sockaddr_in6);
}

// 返回ip地址端口号
int mill_ipport(ipaddr addr) {
    return ntohs(mill_ipfamily(addr) == AF_INET ? 
                        ((struct sockaddr_in*)&addr)->sin_port : ((struct sockaddr_in6*)&addr)->sin6_port);
}

// 转换二进制格式的ip地址为点分ipv4或者ipv6地址格式
const char *mill_ipaddrstr_(ipaddr addr, char *ipstr) {
    if (mill_ipfamily(addr) == AF_INET) {
        return inet_ntop(AF_INET, &(((struct sockaddr_in*)&addr)->sin_addr),
            ipstr, INET_ADDRSTRLEN);
    }
    else {
        return inet_ntop(AF_INET6, &(((struct sockaddr_in6*)&addr)->sin6_addr),
            ipstr, INET6_ADDRSTRLEN);
    }
}

// 获取本地ip地址
// 二进制格式地址，name可以为空、可以为ipv4或ipv6点分表示形式、可以为接口名称
ipaddr mill_iplocal_(const char *name, int port, int mode) {
    if(!name)
        return mill_ipany(port, mode);
    ipaddr addr = mill_ipliteral(name, port, mode);
#if defined __sun
    return addr;
#else
    if(errno == 0)
       return addr;
    /* Address is not a literal. It must be an interface name then. */
    struct ifaddrs *ifaces = NULL;
    int rc = getifaddrs (&ifaces);
    mill_assert (rc == 0);
    mill_assert (ifaces);
    /*  Find first IPv4 and first IPv6 address. */
    struct ifaddrs *ipv4 = NULL;
    struct ifaddrs *ipv6 = NULL;
    struct ifaddrs *it;
    for(it = ifaces; it != NULL; it = it->ifa_next) {
        if(!it->ifa_addr)
            continue;
        if(strcmp(it->ifa_name, name) != 0)
            continue;
        switch(it->ifa_addr->sa_family) {
        case AF_INET:
            mill_assert(!ipv4);
            ipv4 = it;
            break;
        case AF_INET6:
            mill_assert(!ipv6);
            ipv6 = it;
            break;
        }
        if(ipv4 && ipv6)
            break;
    }
    /* Choose the correct address family based on mode. */
    switch(mode) {
    case IPADDR_IPV4:
        ipv6 = NULL;
        break;
    case IPADDR_IPV6:
        ipv4 = NULL;
        break;
    case 0:
    case IPADDR_PREF_IPV4:
        if(ipv4)
           ipv6 = NULL;
        break;
    case IPADDR_PREF_IPV6:
        if(ipv6)
           ipv4 = NULL;
        break;
    default:
        mill_assert(0);
    }
    if(ipv4) {
        struct sockaddr_in *inaddr = (struct sockaddr_in*)&addr;
        memcpy(inaddr, ipv4->ifa_addr, sizeof (struct sockaddr_in));
        inaddr->sin_port = htons(port);
        freeifaddrs(ifaces);
        errno = 0;
        return addr;
    }
    if(ipv6) {
        struct sockaddr_in6 *inaddr = (struct sockaddr_in6*)&addr;
        memcpy(inaddr, ipv6->ifa_addr, sizeof (struct sockaddr_in6));
        inaddr->sin6_port = htons(port);
        freeifaddrs(ifaces);
        errno = 0;
        return addr;
    }
    freeifaddrs(ifaces);
    ((struct sockaddr*)&addr)->sa_family = AF_UNSPEC;
    errno = ENODEV;
    return addr;
#endif
}

// 获取远程机器ip地址的二进制格式
// name可以是ipv4或ipv6点分表示形式、域名domain（这里要用到dns查询）
ipaddr mill_ipremote_(const char *name, int port, int mode, int64_t deadline) {
    int rc;
    ipaddr addr = mill_ipliteral(name, port, mode);
    if(errno == 0)
       return addr;
    /* Load DNS config files, unless they are already chached. */
    if(mill_slow(!mill_dns_conf)) {
        /* TODO: Maybe re-read the configuration once in a while? */
        mill_dns_conf = dns_resconf_local(&rc);
        mill_assert(mill_dns_conf);
        mill_dns_hosts = dns_hosts_local(&rc);
        mill_assert(mill_dns_hosts);
        mill_dns_hints = dns_hints_local(mill_dns_conf, &rc);
        mill_assert(mill_dns_hints);
    }
    /* Let's do asynchronous DNS query here. */
    struct dns_resolver *resolver = dns_res_open(mill_dns_conf, mill_dns_hosts,
        mill_dns_hints, NULL, dns_opts(), &rc);
    mill_assert(resolver);
    mill_assert(port >= 0 && port <= 0xffff);
    char portstr[8];
    snprintf(portstr, sizeof(portstr), "%d", port);
    struct addrinfo hints;
    memset(&hints, 0, sizeof(hints));
    hints.ai_family = PF_UNSPEC;
    struct dns_addrinfo *ai = dns_ai_open(name, portstr, DNS_T_A, &hints,
        resolver, &rc);
    mill_assert(ai);
    dns_res_close(resolver);
    struct addrinfo *ipv4 = NULL;
    struct addrinfo *ipv6 = NULL;
    struct addrinfo *it = NULL;
    while(1) {
        rc = dns_ai_nextent(&it, ai);
        if(rc == EAGAIN) {
            int fd = dns_ai_pollfd(ai);
            mill_assert(fd >= 0);
            int events = fdwait(fd, FDW_IN, deadline);
            /* There's no guarantee that the file descriptor will be reused
               in next iteration. We have to clean the fdwait cache here
               to be on the safe side. */
            fdclean(fd);
            if(mill_slow(!events)) {
                errno = ETIMEDOUT;
                return addr;
            }
            mill_assert(events == FDW_IN);
            continue;
        }
        if(rc == ENOENT)
            break;

        if(!ipv4 && it && it->ai_family == AF_INET) {
            ipv4 = it;
        }
        else if(!ipv6 && it && it->ai_family == AF_INET6) {
            ipv6 = it;
        }
        else {
            free(it);
        }

        if(ipv4 && ipv6)
            break;
    }
    switch(mode) {
    case IPADDR_IPV4:
        if(ipv6) {
            free(ipv6);
            ipv6 = NULL;
        }
        break;
    case IPADDR_IPV6:
        if(ipv4) {
            free(ipv4);
            ipv4 = NULL;
        }
        break;
    case 0:
    case IPADDR_PREF_IPV4:
        if(ipv4 && ipv6) {
            free(ipv6);
            ipv6 = NULL;
        }
        break;
    case IPADDR_PREF_IPV6:
        if(ipv6 && ipv4) {
            free(ipv4);
            ipv4 = NULL;
        }
        break;
    default:
        mill_assert(0);
    }
    if(ipv4) {
        struct sockaddr_in *inaddr = (struct sockaddr_in*)&addr;
        memcpy(inaddr, ipv4->ai_addr, sizeof (struct sockaddr_in));
        inaddr->sin_port = htons(port);
        dns_ai_close(ai);
        free(ipv4);
        errno = 0;
        return addr;
    }
    if(ipv6) {
        struct sockaddr_in6 *inaddr = (struct sockaddr_in6*)&addr;
        memcpy(inaddr, ipv6->ai_addr, sizeof (struct sockaddr_in6));
        inaddr->sin6_port = htons(port);
        dns_ai_close(ai);
        free(ipv6);
        errno = 0;
        return addr;
    }
    dns_ai_close(ai);
    ((struct sockaddr*)&addr)->sa_family = AF_UNSPEC;
    errno = EADDRNOTAVAIL;
    return addr;
}
```

