---
layout:     post
title:      谈谈 Kong rate-limiting 插件中的缺陷
subtitle:   Redis 高频卡控中的 Race Conditions 问题
date:       2018-04-04
author:     ms2008
header-img: img/post-bg-kongtribuitor-shirt.png
catalog:    true
tags:
    - Kong
    - Atomicity
    - Redis
    - Race Conditions
typora-root-url: ..
---

知名 API 网关 [Kong][1] 有个 [rate-limiting][2] 的插件，可以利用它来实现限流的需求。例如：根据特定时间窗口来限制 API 的调用次数。其关键代码是这么实现的：

```lua
red:init_pipeline()
for i = 1, idx do
  red:incrby(keys[i], value)
  if expirations[i] then
    red:expire(keys[i], expirations[i])
  end
end

local _, err = red:commit_pipeline()
if err then
  ngx_log(ngx.ERR, "failed to commit pipeline in Redis: ", err)
  return nil, err
end
```

看上去逻辑非常简单，然而这里却有个陷阱：**无法保证请求的原子性**。即，当有大量的请求到达时，`expire` 可能会执行多次，导致过期时间会被多次刷新，进而导致「KEY」的过期时间会被拉长（*然而这里却意外得到一个好处，继续往下看*）。

另外一个问题是：**KEY 的时效性，也就是 TTL**。Kong 是严格按照业务需求来定义的：

```lua
local EXPIRATIONS = {
  second = 1,
  minute = 60,
  hour = 3600,
  day = 86400,
  month = 2592000,
  year = 31536000,
}
```

我们这里来试想这么一个场景：

![](/img/in-post/redis-rate-limit.png)

如上，我们现在有一个 KEY，TTL 设置为 1s，而现在距离 KEY 过期还剩下 200 ms。假设，现在有一个请求到 Redis 往返需要 600 ms，也就是说单趟需要耗时 300 ms 左右。那么将会发生：<u>一直等到这个 KEY 过期之后，请求才到达 Redis，于是 Redis 会重新创建这个**同名的 KEY**并返回 1 给请求，导致卡控失效</u>。

解决这个问题也很简单，就是要保证 TTL 的时效要大于限制的周期。一个完整的实现可以参考下面这样：

```js
local PERIOD = 1 -- 1s
local EXPIRATION = 60 -- 60s

local key = ngx.var.uri .. ":" .. (math.floor(ngx.time() / PERIOD))
local red = redis:new()

local counter, err = red:eval([[
    local tally = redis.call('INCR', ARGV[1])

    if tally == 1 then
        redis.call('EXPIRE', ARGV[1], ARGV[2])
    end

    return tally
]], 0, key, EXPIRATION)

if not counter then
    ngx.say("eval error: ", err)
    return
end

if tonumber(counter) <= limit then
    return
else
    ngx.say("API rate limit exceeded")
end
```

最后，Kong 的 Blog 上也总结了几种限流方案，感兴趣的可以去瞅瞅 👉 [How to Design a Scalable Rate Limiting Algorithm][3]

[1]: https://getkong.org/
[2]: https://getkong.org/plugins/rate-limiting/
[3]: https://konghq.com/blog/how-to-design-a-scalable-rate-limiting-algorithm/
