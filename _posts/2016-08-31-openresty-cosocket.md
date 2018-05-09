---
layout:     post
title:      cosocket
subtitle:   cosocket 的 connect 是会有创建 sock 的行为的
date:       2016-08-31
author:     ms2008
header-img: img/post-bg-re-vs-ng2.jpg
catalog:    true
tags:
    - OpenResty
    - cosocket
typora-root-url: ..
---

> TCP 协议是面向流的。面向流是指无保护消息边界的，如果发送端连续发送数据，接收端有可能在一次接收动作中会接收两个或者更多的数据包。

在传统的 socket 编程中 `socket.socket()` 会创建一个套接字(内核对象)，本质就是一个 socket 文件句柄(套接字句柄)，和普通的文件差不多。只有在调用 `connect()` 或 `bind()` 之后才会产生可以网络通讯的套接字(给套接字赋予地址)。一个 connection 本质上就是一个 sock 文件，连接池的本质也就是保持这个 sock 文件。

> “ `socket.socket()` 会创建一个套接字(内核对象)”，这个描述可能还不是非常准确。把其理解为引用更为恰当些，因为真正的创建是在绑定五元组的时候，也就是 connect 操作之后。。

但是对于 openresty 中的 cosocket，socket 是封装了一套同步非阻塞的 API 的对象(此 socket 非彼 sock)，sock 才是内核级别的 socket 套接字句柄(也就是连接)。其中的 `connect()` 方法和传统 socket 编程中的 `accept()` 一样，也是会生成新的 sock 对象的，前提是 `ngx.socket.tcp()` 已经空闲。

Example:

```
./resty -e '
   local sock = ngx.socket.tcp()
   ngx.say(sock:connect("127.0.0.1", 80))
   ngx.say(sock:setkeepalive())

   ngx.say(sock:connect("127.0.0.1", 8989))
   ngx.say(sock:setkeepalive())

   ngx.sleep(600)
'

./resty -e '
   local sock = ngx.socket.tcp()
   ngx.say(sock:connect("127.0.0.1", 80))
   ngx.say(sock:connect("127.0.0.1", 8989))

   ngx.say(sock:setkeepalive())
   ngx.sleep(600)
'
```

```sh
netstat -anp | grep -w '80\|8989'
ss -anp dport = :80 or dport = :8989| cat
```

![](/img/in-post/cosocket.jpg)

可以看到在 /proc/7619/fd 下，确实是 2个 sock
