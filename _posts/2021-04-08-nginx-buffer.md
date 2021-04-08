---
layout:     post
title:      "Nginx Buffer 机制引发的下载故障"
subtitle:   "Nginx Buffer 机制研究"
date:       2021-04-08
author:     "ms2008"
header-img: "img/post-bg-os-metro.jpg"
catalog:    true
tags:
    - Nginx
typora-root-url: ..
---

前几天，接到研发同事反馈，内网通过 Nginx 代理下载 OSS 的大文件时，老是会断，而在服务器上下载时却很正常，非常奇怪。原本以为可能和 VPN 有关，经确认排除嫌疑。彷徨了许久，最后发现是 Nginx Buffer 的锅。下面就来聊聊这个问题是怎么发生的。

## Nginx Buffer 机制

在 Nginx 代理过程中，有两种连接：

1. 客户端到 Nginx 的连接
2. Nginx 到后端服务器的连接

这两个连接的速度不一致，会对客户端的体验带来不良的影响。Nginx 正是通过 Buffer 机制来缓解这个问题。

当启用 buffer 时，Nginx 将会临时存储后端响应内容在内存或者磁盘上，然后慢慢把数据推送给客户端；若关闭，则会按照响应内容的多少立刻同步到客户端。

假设客户端的速度足够快，那么完全可以把 buffer 关掉，让数据尽可能快速到达；如果客户端很慢，就应该保持 buffer 开启，这样有利于 Nginx 和后端的连接复用（本质上是 HTTP 队头阻塞问题）。

## 大文件下载问题

Nginx Buffer 机制默认处于开启状态，其会根据 `proxy_buffer_size` 和 `proxy_buffers` 这两个参数控制写入内存的大小。如果响应大于这个 buffer 大小，Nginx 会继续通过 `proxy_max_temp_file_size` 参数将响应其余部分写入到磁盘临时文件。

那么问题来了，如果响应还是很大，超过了临时文件的限额怎么办？

**等！** <u>此时，Nginx 的 socket 缓冲区也是出于满载状态。由于客户端很慢，Nginx 并没有触发 `read` 后端操作。这里大概率会触发后端服务器的 `write` 超时，进而由后端发起 `close` 操作。</u>

这就是我遇到的问题：`proxy_max_temp_file_size` 默认为 1G，当客户端的网络比较慢时，临时文件很快就被写满。这时候后端的响应还会继续被接收到 socket 缓冲区，直到缓冲区被打满。此时，Nginx 所在服务器通过滑动窗口 zero 0 告知后端服务器停止发送数据，直至触发了后端的 `write` 超时。

![](/img/in-post/wget-partially-downloaded.png)

而当客户端的网络比较快时，临时文件并不会被写满，或者即使写满了也很快就会消费掉，不至于让后端“阻塞”过长时间触发超时。

## 如何解决？

1. 调整 `proxy_max_temp_file_size` 大小

    - 调大

      让临时文件足够可以缓冲整个响应

    - 调小

      让整个链路上的数据流动起来，不要阻塞后端的 `write` 操作，进而触发后端的超时

2. 关闭 Buffer

   不推荐，会影响 Nginx 到后端的连接复用

## 参考文章

- [Module ngx_http_proxy_module](http://nginx.org/en/docs/http/ngx_http_proxy_module.html)
- [Nginx —— 理解HTTP 代理，负载均衡，缓冲（Buffering）和缓存（Caching）](https://n3xtchen.github.io/n3xtchen/nginx/2016/02/19/nginx-port-forwording)
