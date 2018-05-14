---
layout:     post
title:      浅谈 Kong key-auth 插件 token 的生成
subtitle:   CSPRNG or 真随机？
date:       2018-04-16
author:     ms2008
header-img: img/post-bg-kongtribuitor-shirt.png
catalog:    true
tags:
    - Kong
    - PRNG
    - Random
---

最近我在 Kong 的 Blog 上看了一篇文章：[That’s So Random: (Pseudo)Random Data Generation in Kong API Gateway][1]，文章中介绍了 Kong 是怎么处理随机数问题的，读后受益良多，在此做一个分享。

### seed 的生成

在 OpenResty 中如果使用 `ngx.now()` 设置种子的话，将会导致各个 worker 的种子相同，也就是说每个 worker 的随机性其实是一样的。一个优化的方案是 `ngx.now()*1000 + ngx.worker.pid()`, 但是在分布式的环境中，这样依然会有一定的概率产生相同的种子。

Kong 的解决方案是利用 OpenSSL 的 `RAND_bytes()` 来生成种子。具体方法是：先读取 8 个字节，之后按每个字节做 `byte` 操作，再用 `concat` 连接起来。由于 Lua 的 number 其实是 double float，小数有效位是 15-16 位，为了防止其越界，KONG 只取了其前 12 位做为种子。具体实现如下：

```lua
local bytes, err = util.get_rand_bytes(8)
if bytes then
  ngx.log(ngx.DEBUG, "seeding PRNG from OpenSSL RAND_bytes()")

  local t = {}
  for i = 1, #bytes do
    local byte = string.byte(bytes, i)
    t[#t+1] = byte
  end
  local str = table.concat(t)
  if #str > 12 then
    -- truncate the final number to prevent integer overflow,
    -- since math.randomseed() could get cast to a platform-specific
    -- integer with a different size and get truncated, hence, lose
    -- randomness.
    -- double-precision floating point should be able to represent numbers
    -- without rounding with up to 15/16 digits but let's use 12 of them.
    str = string.sub(str, 1, 12)
  end
  seed = tonumber(str)
else
  ngx.log(ngx.ERR, "could not seed from OpenSSL RAND_bytes, seeding ",
                   "PRNG with time and worker pid instead (this can ",
                   "result to duplicated seeds): ", err)

  seed = ngx.now()*1000 + ngx.worker.pid()
end
```

### token 的生成

早期的 Kong 生成 token 用的是 UUID，去掉 `-` 连字符，是一个 32 位长的字符串。但是其 UUID 生成依赖的 LuaJIT 的 PRNG，并不属于 CSPRNG，所以不适合这一类对安全要求比较高的场景。目前 KONG 用的是系统的 `urandom`，可以认为是一个真随机的实现。相关实现如下：

```lua
local function urandom_bytes(buf, size)
  local fd = ffi.C.open("/dev/urandom", O_RDONLY, 0) -- mode is ignored
  if fd < 0 then
    ngx_log(WARN, "Error opening random fd: ",
                  ffi_str(ffi.C.strerror(ffi.errno())))

    return false
  end

  local res = ffi.C.read(fd, buf, size)
  if res <= 0 then
    ngx_log(WARN, "Error reading from urandom: ",
                  ffi_str(ffi.C.strerror(ffi.errno())))

    return false
  end

  if ffi.C.close(fd) ~= 0 then
    ngx_log(WARN, "Error closing urandom: ",
                  ffi_str(ffi.C.strerror(ffi.errno())))
  end

  return true
end

local function random_string()
  return encode_base64(get_rand_bytes(24, true))
         :gsub("/", char(rand(48, 57)))  -- 0 - 10
         :gsub("+", char(rand(65, 90)))  -- A - Z
         :gsub("=", char(rand(97, 122))) -- a - z
end
```

但是这样有个缺点就是会阻塞 worker，至于为什么不用性能更好的 OpenSSL’s CSPRNG。Kong 也给出了解释：

- 目前 OpenSSL’s RNG 的被发现有一些缺陷，可能会在未来修复。当下对于 KONG 来说，使用内核的 CSPRNG 无疑是最好的选择。
- 生成 token 的动作由 Kong admin 发起，并不会很频繁，而且阻塞时间很短是可以接受的。

[1]: https://konghq.com/blog/pseudorandom-data-generation-in-kong-api-gateway/
