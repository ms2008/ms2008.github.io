---
layout:     post
title:      OpenResty Coroutine 的调度机制
subtitle:   ""
date:       2016-07-06
author:     ms2008
header-img: img/home-bg-geek.jpg
catalog:    true
tags:
    - OpenResty
    - Lua
    - Coroutine
---

在聊这个话题之前，我们需要明确的是 Nginx 的多进程单线程的进程模型。OpenResty 正是基于其 master → worker 模型，在 master fork 出 worker 时，将存在于 master 中的 lua vm 通过系统的 cow 机制传递到 worker 中。

> 注：既然是通过 fork 传递过去的，所以 `init_by_lua` 里面创建的 Lua 全局变量对各个 worker 进程而言只有只读的价值

worker 抢到的每个外部请求都是 ngx_lua 的一个 coroutine，coroutine 之间数据隔离（全局变量每次都是新的）。worker 内所有 coroutine 共享一个 vm。

> 注：Lua 的全局变量的局部化，在 ngx_lua 中是通过为协程对应的主 Lua 函数设置独立的环境表来实现状态隔离的（即通过 lua_setfenv 调用）。但是每个请求的全局变量表同时又会通过 lua 的 __index 元方法自动继承 init_by_lua 里的真正的全局变量表。这就是为什么在 init_by_lua 上下文中创建的 Lua 全局变量对于所有请求的上下文都是可见的原因。

理解了以上内容，我们再来看 ngx_lua 对于协程的调度：coroutine 如果没有做I/O 操作和 sleep，会一直占用 lua vm，直到这个请求结束。ngx_lua 的 coroutine 调度机制是遇到 I/O 和 sleep 就让出 CPU，其他情况不会让出 CPU （也并不能严格的说遇到 I/O 就会让出 CPU，比如数据都已经在内核缓存区准备好了，此时 `cosocket:read` 也不会让出 CPU）。这一点其实在 github 的官方文档就有提及，只是中文版的 wiki 并没有表达明白春哥的意思：

> You can also share changeable data among all the concurrent requests of each nginx worker process as long as there is no nonblocking I/O operations (including ngx.sleep) in the middle of your calculations. As long as you do not give the control back to the nginx event loop and ngx_lua's light thread scheduler (even implicitly), there can never be any race conditions in between.

如果用一句话来概括就是：请求过程是否会有 `yield` （cosocket 和 `ngx.sleep`）的操作。正是因为如此，所以必须小心处理一些热逻辑，否则会导致整个 worker 被占用（相当于被阻塞了）：

```lua
while true do
    -- forget to break
end
```

在 ngx_lua 中对共享 Lua 变量的读写操作序列，只要中间没有涉及 IO 操作（或者类似 `ngx.sleep` 这样的操作），就一定是原子的。因为：

1. Nginx worker 进程是单 OS 线程的
2. ngx_lua 的“轻量级线程”是协作式的，而非抢占式的

模块是不会有 race condition 问题的（这也是 lua 非抢占式协程的好处，但是可能会“串”）

值得一提的是，nginx 的多 worker 进程模型并不像 php-fpm 那样，即并不是一个 worker 进程对应一个下游连接。nginx 的多 worker 进程只是为了用满多个 CPU 核而已，每个 worker 进程可以处理很多的并发连接。ngx_lua 的连接池和 nginx 核心中针对 upstream 模块的连接池一样是在每一个 worker 进程的级别上由许多并发连接共享的

---

参考文档：

- [OpenResty Google Groups](https://groups.google.com/forum/#!forum/openresty)
- [Github openresty/lua-nginx-module](https://github.com/openresty/lua-nginx-module)
