---
layout:     post
title:      捋一捋 backlog 的作用
subtitle:   聊不完的 TCP/IP
date:       2016-09-06
author:     ms2008
header-img: img/post-bg-universe.jpg
catalog:    true
tags:
    - TCP
    - backlog
typora-root-url: ..
---

我们知道在 socket 编程中，服务端需要经历 listen → bind → accept 这么几个过程；而客户端需要经历 connect → receive 的过程。其中服务端在 bind 的时候需要指定 backlog 的大小。网上的好多文章，感觉都没有讲清楚这个参数的作用，特在此好好捋一下。方便他人，同时也方便自己。

术语约定：

- 未完成：半开，处于 SYN_RCVD 状态
- 已完成：已连接但未 ACCEPT，处于 ESTABLISHED 状态
- 已完毕：已连接且已 ACCEPT，处于 ESTABLISHED 状态

以下我们这里只讨论未完成和已完成，因为已完毕的都是由应用自己维护。直接上结论：

```
linux 2.2 之前：
    backlog = 未完成 + 已完成

linux 2.2 之后：
    backlog = 已完成
```

需要注意的是，linux 2.2 之后，半连接 syn 队列的长度由 `max(64, /proc/sys/net/ipv4/tcp_max_syn_backlog)` 所决定；而已完成的队列长度其实是由 `min(backlog, somaxconn)` 所决定。`somaxconn` 这个值，在 kernel 2.4.25 之前，都是写死在代码里的，默认为 128。在 kernel 2.4.25 之后，在 `/proc/sys/net/core/somaxconn` 。

那么问题来了，怎样才能查看进程的 backlog 数量呢？执行：

```sh
ss -anp | grep nginx
```

就可以了。需要注意的是，使用该命令获取到的 Recv-Q/Send-Q 在 LISTEN 状态以及非 LISTEN 状态所表达的含义是不同的：

![](/img/in-post/backlog.png)

- LISTEN 状态

  Recv-Q 表示的是当前已完成队列的大小；Send-Q 表示的则是最大的 listen backlog 数值。

- 其余状态

  Recv-Q 表示 receive queue 中的 bytes 数量；Send-Q 表示 send queue 中的 bytes 数值。

所以上图中显示的是，nginx 的 `backlog` 为 511，当前已完成队列的大小为 0。那么还有个问题，就是未完成队列在哪里看？很简单：

```sh
ss -anp | grep nginx | grep -c SYN_RECV
```
