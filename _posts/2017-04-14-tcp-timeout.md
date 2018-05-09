---
layout:     post
title:      理解 timeout，这一篇就够了
subtitle:   对 TCP 协议栈的理解总是需要慢慢积累
date:       2017-04-14
author:     ms2008
header-img: img/post-bg-e2e-ux.jpg
catalog:    true
tags:
    - TCP
    - Nginx
typora-root-url: ..
---

PS. 最近比较忙，原先的 Lua 排序算法系列暂时搁置一段时间，以后还是会接着更新的。要是我精力足够，将来可能还会出 Lua 搜索算法系列，尽请期待。

嗯，好吧，有些标题党了。不过这个问题我在 google 了大量中文的资料后，都没能解释清楚我的疑惑。索性深入研究了几天，自己来总结下学习成果。

----

如果你的 Nginx 服务器流量足够大，足够繁忙。可能你会在 Nginx 的 error log 中看到下面这样的日志：

![Alt text](/img/in-post/nginx_timeout.png)

如果你去仔细观察，会发现这两个的 timeout 似乎有些不太一样：

> - 110: Connection timed out
> - timeout

那这两种 timeout 有什么区别？分别在什么情况下会发生？

首先无论是哪种语言，不管是客户端还是服务端，在 TCP 编程中通常都可以为 sock 设置一个 timeout 时间。而这个 timeout 又可以细分为 connect timeout、read timeout、write timeout。read timeout 和 write timeout 必须是在 connect 之后才能发生，今天不做过多讨论。上面那两种 timeout 均属于 connect timeout。

另外我们需要补充下 TCP 重传机制的相关知识：

我们知道在 TCP 的三次握手中，Client 发送 SYN，Server 收到之后回 SYN_ACK，接着 Client 再回 ACK，这时 Client 便完成了 connect() 调用，进入 ESTAB 状态。如果 Client 发送 SYN 之后，由于网络原因或者其他问题没有收到 Server 的 SYN_ACK，那么这时 Client 便会重传 SYN。重传的次数由内核参数 `net.ipv4.tcp_syn_retries` 控制，重传的间隔为 [1,3,7,15,31]s 等。(有很多文章都说是 [3,6,9]s 等，不过根据我的抓包分析，貌似不太符合，如下图)

![Alt text](/img/in-post/syn_retry.png)

**如果 Client 重传完所有 SYN 之后依然没有收到 SYN_ACK，那么这时 connect() 调用便会抛出 connection timeout 错误。如果 Client 在重传 SYN 期间，Client 的 sock timeout 时间到了，那么这时 connect() 会抛出 timeout 错误。**

下面是我用 Lua 写的测试用例，可以看到我为 sock 设置了 60s 的超时时间：

```lua
local red = redis:new()
red:set_timeout(60000)

-- timeout test
local ok, err = red:connect("8.8.8.8", 6379)
if not ok then
    ngx.log(ngx.ERR, "failed to connect: ", err)
    return
end
```

先来复现第一种情况

#### 110: Connection timed out

我先把 `net.ipv4.tcp_syn_retries` 置为 1，也就是说现在会重传1次 SYN，需要等待 SYN_ACK 的时间为 3s。
![Alt text](/img/in-post/connection_timeout_1.png)
![Alt text](/img/in-post/connection_timeout_2.png)

可以看到符合预期

再来复现另外一种情况

#### timeout

这回把 `net.ipv4.tcp_syn_retries` 置为 5，也就是说现在会重传5次 SYN，需要等待 SYN_ACK 的时间为 63s。
![Alt text](/img/in-post/connect_timeout_1.png)
![Alt text](/img/in-post/connect_timeout_2.png)

Bingo!

----

##### 参考资料

* [Overriding the default Linux kernel 20-second TCP socket connect timeout](http://www.sekuda.com/overriding_the_default_linux_kernel_20_second_tcp_socket_connect_timeout)
* [MySQL Connection Timeouts](https://www.percona.com/blog/2011/04/19/mysql-connection-timeouts/)
