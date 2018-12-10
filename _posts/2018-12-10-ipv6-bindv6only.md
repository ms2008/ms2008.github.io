---
layout:     post
title:      "IPv4 也是可以访问 IPv6 服务的"
subtitle:   "Golang 总是喜欢一厢情愿的隐藏掉很多细节"
date:       2018-12-10
author:     "ms2008"
header-img: "img/post-bg-golang-blog-hero.jpg"
catalog:    true
tags:
    - TCP
    - Golang
typora-root-url: ..
---

## 起因

对于 Golang 的 `net.Listen()` 函数，如果你不强行指定 IPv4 或 IPv6 的话，**在双栈系统上默认只会监听 IPv6 地址**。比如，用 Golang 实现一个 HTTP 服务非常简单：

```go
package main

import (
	"net/http"
)

type helloHandler struct{}

func (h *helloHandler) ServeHTTP(w http.ResponseWriter, r *http.Request) {
	w.Write([]byte("Hello, world!"))
}

func main() {
	http.Handle("/", &helloHandler{})
	http.ListenAndServe(":6666", nil)
}
```

启动后，通过 `netstat` 查看可以确定服务只有在 IPv6 上监听：

![](/img/in-post/ipv6-bindv6only/netstat.png)

这时候，如果 `curl` 的是服务的 IPv4 地址，其实也是可以看到预期输出的：

```
$ curl -i -s http://192.168.3.10:6666
HTTP/1.1 200 OK
Date: Sat, 08 Dec 2018 06:33:54 GMT
Content-Length: 13
Content-Type: text/plain; charset=utf-8

Hello, world!
```

之所以会有这样的行为，**是因为在 linux 上有个内核参数 `net.ipv6.bindv6only` 默认为关闭状态，这样 IPv6 的 socket 也就可以解析映射到同一个网卡的 IPv4 请求了**。这样的话，如果我们的服务需要同时提供 IPv4 和 IPv6 的访问能力，只需要监听一个 IPv6 的 socket 即可。

我这里并不希望 IPv4 可以访问 IPv6 的服务，所以我把 `net.ipv6.bindv6only` 置为了 1：

```
$ cat /proc/sys/net/ipv6/bindv6only
1
```

遗憾的是，开启了这个参数后，似乎并没有效果，`curl` 依然可以使用 IPv4 的地址来访问。

## 溯源

怎么解释这个问题？既然是看起来内核参数没有生效，问题多半是发生在 `syscall` 的调用上，祭出 `strace` 大杀器：

![](/img/in-post/ipv6-bindv6only/strace.png)

<u>可以看到在 220 行 Golang 把准备 `listen` 的 socket 选项置为了 0。显然这个是 Golang 自身的行为</u>。

通过追踪 Golang 的源码，问题最终定位在了这里 `src/net/ipsock_posix.go` ：

```go
func favoriteAddrFamily(network string, laddr, raddr sockaddr, mode string) (family int, ipv6only bool) {
    switch network[len(network)-1] {
    case '4':
        return syscall.AF_INET, false
    case '6':
        return syscall.AF_INET6, true
    }

    if mode == "listen" && (laddr == nil || laddr.isWildcard()) {
        if supportsIPv4map() || !supportsIPv4() {
            return syscall.AF_INET6, false
        }
        if laddr == nil {
            return syscall.AF_INET, false
        }
        return laddr.family(), false
    }

    if (laddr == nil || laddr.family() == syscall.AF_INET) &&
        (raddr == nil || raddr.family() == syscall.AF_INET) {
        return syscall.AF_INET, false
    }
    return syscall.AF_INET6, false
}
```

原来 Golang 自己定义了 IPV6_V6ONLY 这个行为，至于这么做的原因在官方 Github 也有一些讨论：[net: Listen is unfriendly to multiple address families, endpoints and subflows][1]

既然是这样，那如何解决这个问题？

- 自己定制需要的 `net.Listen()`
- `listen` 完整的 IPv4 和 IPv6 地址

## 插曲

如果你用 Chrome 访问 http://localhost:6666 这样的地址，可能会看到 `ERR_UNSAFE_PORT` 这样的错误页面。这个其实是因为 Chrome 的非安全端口限制。

![](/img/in-post/ipv6-bindv6only/chrome.png)

像这样的端口，一共有 64 个：

```go
1,    // tcpmux
7,    // echo
9,    // discard
11,   // systat
13,   // daytime
15,   // netstat
17,   // qotd
19,   // chargen
20,   // ftp data
21,   // ftp access
22,   // ssh
23,   // telnet
25,   // smtp
37,   // time
42,   // name
43,   // nicname
53,   // domain
77,   // priv-rjs
79,   // finger
87,   // ttylink
95,   // supdup
101,  // hostriame
102,  // iso-tsap
103,  // gppitnp
104,  // acr-nema
109,  // pop2
110,  // pop3
111,  // sunrpc
113,  // auth
115,  // sftp
117,  // uucp-path
119,  // nntp
123,  // NTP
135,  // loc-srv /epmap
139,  // netbios
143,  // imap2
179,  // BGP
389,  // ldap
465,  // smtp+ssl
512,  // print / exec
513,  // login
514,  // shell
515,  // printer
526,  // tempo
530,  // courier
531,  // chat
532,  // netnews
540,  // uucp
556,  // remotefs
563,  // nntp+ssl
587,  // stmp?
601,  // ??
636,  // ldap+ssl
993,  // ldap+ssl
995,  // pop3+ssl
2049, // nfs
3659, // apple-sasl / PasswordServer
4045, // lockd
6000, // X11
6665, // Alternate IRC [Apple addition]
6666, // Alternate IRC [Apple addition]
6667, // Standard IRC [Apple addition]
6668, // Alternate IRC [Apple addition]
6669, // Alternate IRC [Apple addition]
```

[1]: https://github.com/golang/go/issues/9334
