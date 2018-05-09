---
layout:     post
title:      OpenResty 中的连接池
subtitle:   连接池常见问题排查
date:       2016-09-12
author:     ms2008
header-img: img/post-bg-unix-linux.jpg
catalog:    true
tags:
    - OpenResty
    - connection-pool
---

> 注：set_keepalive 和 close 互斥(一个 socket 对象不能执行多次 setkeepalive 操作，会报：连接已关闭)

连接池的大小是对每一个 nginx worker 而言的。如果有 N 个 worker，最多就会有 N * pool_size 个连接。比如设置 `keepalive = 100`，开始启动时候是 0 连接来一个请求，获取一个 socket(空闲的)，完成请求后，socket<100 时候，就把 socket 放到池子里。每当新请求来的时候，先去池子里面获取空闲 socket，如果获取不到就创建新连接，完成请求后，新连接的数量如果超过设置的 keepalive 值就不放池子里面了。当并发连接数大大超过了连接池的容量，就会产生很多的短连接，此时就应该增加连接池的大小。

连接池常见问题及调试方法：

1. 在测试时配置了 `lua_code_cache off`，即禁用了 Lua 代码缓存，此时连接池的生命期是每请求的，因为 cosocket 连接池是在 Lua VM 里面分配的，而在禁用 Lua 代码缓存时，Lua VM 是每请求一个实例。ngx_drizzle 模块的连接池实现并不受 Lua 代码缓存的影响（出于显而易见的原因）。调试的办法是启用 `lua_code_cache`（默认开启）。

2. 调用 `set_keepalive()` 方法时传入的 max_idle 参数较短（比如示例代码里的 10 秒），这意味着空闲连接在连接池里只会最久逗留这么长的时间，超过就被自动关闭（除非在此之前又为其他请求所复用）。于是你手工在命令行上先运行 ab，再运行 `netstat` 命令时，这中间的时间窗口很容易超出 `max_idle` 限制。而 ngx_drizzle 模块并没有 max idle 保护这个功能。调试的办法是使用 0 作为 `max_idle`，让连接池里的空闲连接永不过期（除非 MySQL server 自己主动关闭或者 nginx 进程退出）。

3. 使用的 MySQL 查询会返回多个结果集，而默认 `query()` 方法只会读取并返回第一个结果集，你需要在 `query()` 方法返回 "again" 这个错误字符串时，自己继续调用 `read_result()` 方法，直至不再返回 "again" 错误串。如果你在没有读取所有结果集之前就调用 `set_keepalive()` 方法，连接池一般会拒绝接收这样的连接，因为该连接的读取缓冲区里还有未读完的数据，此时允许其他会话复用该连接的话，会导致难以调试的错误。调试的办法是，总是对 `set_keepalive()` 的返回值进行恰当的错误处理，并把错误记入 nginx 错误日志。

值得一提的是，nginx 的多 worker 进程模型并不像 Apache prefork mpm 或者 php-fpm 那样，即并不是一个 worker 进程对应一个下游连接。nginx 的多 worker 进程只是为了用满多个 CPU 核而已，每个 worker 进程可以处理很多的并发连接。ngx_lua 的连接池和 nginx 核心中针对 upstream 模块的连接池一样是在每一个 worker 进程的级别上由许多并发连接共享的。
