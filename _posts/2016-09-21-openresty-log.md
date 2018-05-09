---
layout:     post
title:      OpenResty 中写日志的套路
subtitle:   记录日志很重要，套路同样更重要
date:       2016-09-21
author:     ms2008
header-img: img/post-bg-rwd.jpg
catalog:    true
tags:
    - OpenResty
    - log
---

在 OR 中为了方便调试，我们可能会用这样的方式来记录日志：

```lua
local f = io.open("test.log", "a+")
f:write(message .. "\n")
f:close()
```

然而这种写法在大多数情况下，都是可以正常工作的。但是在高并发的系统上，尤其还是多 worker 的环境下，这样写日志是会串的，原因就是 `io.open` 自身的缓存机制(本质是 libc 的缓存机制，因为 lua 的 io 底层还是调用的 `fopen`)。这点春哥也在邮件列表回复过：

> I decided to use “fopen”, “fprintf” and “fclose”:
><br><br>
> As mentioned previously, these libc functions are not safe at all in this context because of its own userland buffering. Use the thin libc wrappers for the syscalls like write() and writev() directly.

那如何解决这个问题？三个解决方案：

1. 使用 `ngx.log`
2. lua-resty-logger-socket
3. 使用 ffi 直接调用 `write()` or `writev()`

需要注意的是，使用第三种解决方案时，到底有没有必要跨 worker 缓存文件句柄？依然引用 agentzh 的回复：

> We don’t need to share the log file descriptor across workers in the first place. Even nginx’s builtin logger does not do this. Thanks to modern operating systems’ atomic file appending feature ;)

最后需要强调的是 `ngx.log` 是输出到错误日志，这个是没有缓冲的，量非常大的话也是极损性能的。nginx 的 access_log 才可以配置缓冲 :)

这一点春哥在微博上都吐槽了：

> 发现一些 nginx 和 openresty 用户在用 ab 这样的工具压测性能时，都不看 nginx 错误日志的。经常因为这些用户自己的配置错误或 Lua 代码错误，而导致 nginx 疯狂地刷错误日志文件。错误日志的输出是文件 I/O，而且还是不带写缓冲的（显然地）。所以他们测出来的性能其实是刷错误日志的性能。无语啊。
