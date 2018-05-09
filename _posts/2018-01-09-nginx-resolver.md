---
layout:     post
title:      NGINX resolver 配置中的 "坑"
subtitle:   不要着急去当第一个吃螃蟹的人
date:       2018-01-09 14:14:10
author:     ms2008
header-img: img/post-bg-e2e-ux.jpg
catalog:    true
tags:
    - Nginx
typora-root-url: ..
---

最近我把自己的 OpenResty 升级到了最新的 `openresty/1.13.6.1` 版本，却发现 dns 解析不能正常工作了：

```lua
...
resolver 127.0.0.1;

server {
    listen       8888;
    server_name  _;

    location / {
        content_by_lua_block {
            local redis = require "resty.redis"

            local red = redis:new()
            local ok, err = red:connect("master", 6379)
            if not ok then
                ngx.say(err)
                ngx.log(ngx.ERR, err)
                return
            end

            local ok, err = red:set("foo", "bar")
            if not ok then
                ngx.say(err)
                ngx.log(ngx.ERR, err)
                return
            end

            local ok, err = red:set_keepalive(60000, 1024)
            if not ok then
                ngx.say(err)
                ngx.log(ngx.ERR, err)
                return
            end

            ngx.say("all seems ok")
        }
    }
}
...
```

在 nginx error.log 可以看到下面的错误：

```
2018/01/08 15:49:14 [error] 10210#0: *1 [lua] content_by_lua(nginx.conf:137):9: master could not be resolved (110: Operation timed out), client: 192.168.3.1, server: localhost, request: "GET / HTTP/1.1", host: "master:8888"
```

但是我通过 `dig @127.0.0.1 master` 确认 DNS 服务器是没有问题的，起初我怀疑是 `openresty/1.13.6.1` 的一个 BUG（因为在 `openresty/1.11.2.4` 版本下是没有这个问题的，并且前些时候我发现这个版本的 `ngx.re` 也存在个 [BUG #1217][2]，让我天真的认为这个版本不太稳定😓），后来通过抓包发现了其中的猫腻：

![](/img/in-post/nginx-resolver.png)

可以看到 DNS 服务器确实响应了 NGINX 的 A 记录查询请求，但是 NGINX 还发出了 AAAA 请求（IPv6 解析请求），不过由于我的 dnsmq 没有启用 IPv6，所以并没有响应它。

后来我在 [Github Issue 504](https://github.com/openresty/lua-nginx-module/issues/504) 找到了关于这个问题的讨论：

> So if my OpenResty is built with the --with-ipv6 option, then the nginx builtin resolver will query both IPv4 and IPv6 DNS records. And each request will pick up one DNS record by chance. When getting an IPv6 address, your connect() call receives a "connection refused" error.

原来当 NGINX 启用了 `--with-ipv6` 选项时，resolver 就会同时查询 IPv4 和 IPv6 的 DNS 记录。NGINX 会随机选一个 DNS 查询请求，之后再去连接对应的地址。

但是从 `nginx 1.11.5` 开始，nginx 取消掉了 `--with-ipv6` 这个参数([参考这里][1])，并且自动启用了 IPv6。所以就发生了上面的异常，解决这个问题也很简单，就是手动关闭掉 resolver 的 IPv6 解析就好：`resolver 127.0.0.1 ipv6=off;`

**我很奇怪 NGINX 在处理 DNS 响应时，为什么不用 [Happy Eyeballs][3] 之类的算法呢？**

[1]: http://nginx.org/en/CHANGES
[2]: https://github.com/openresty/lua-nginx-module/issues/1217
[3]: http://www.rfc-base.org/rfc-6555.html
