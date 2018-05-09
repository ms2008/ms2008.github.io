---
layout:     post
title:      重新认识 randomseed
subtitle:   原生 Lua 的 randomseed 是不支持浮点数的
date:       2017-10-27
author:     ms2008
header-img: img/post-bg-universe.jpg
catalog:    true
tags:
    - Lua
    - LuaJIT
    - OpenResty
    - PRNG
---

在之前的文章中，我写过这样的测试用例：

```lua
local seed = 123456
for i=1,2 do
    math.randomseed(seed + (i-1)/10)
    local num = {}
    for j=1,10 do
        table.insert(num, math.random(100))
    end
    print(table.concat(num, ","))
end
```

运行结果是:

```
61,66,19,75,44,61,68,1,33,4
61,66,19,75,44,61,68,1,33,4
```

目的是为了引出 LCG 的缺点，然而这并不能证明 LCG 算法生成的随机数列质量就不够好。这是因为 `randomseed` 底层调用的是 C 的 `srand` ，其需要传入的是一个 unsigned int。Lua 的 `randomseed` 只是对其做了个包装：如果你传入的不是 int，Lua 会对其进行数值修约，再将修约之后的值传递给 `srand` 调用，**也就是说 `randomseed` 是不支持浮点数的**。上面的用例中，其实 seed 是一样的，都是 123456，因而产生的随机数列是一样的。

另外，需要注意的是由于 `srand` 接受的是一个 int。而 Lua 用的是 double，那么如果你将 `randomseed` 的 seed 越界，那么其随机性将会被打破。例如：

```lua
math.randomseed(2^40)
local num1 = {}
for i=1,10 do
    table.insert(num1, math.random(100))
end
print(table.concat(num1, ","))

math.randomseed(2^52)
local num2 = {}
for i=1,10 do
    table.insert(num2, math.random(100))
end
print(table.concat(num2, ","))

math.randomseed(0)
local num3 = {}
for i=1,10 do
    table.insert(num3, math.random(100))
end
print(table.concat(num3, ","))
```

运行结果:

```
85,40,79,80,92,20,34,77,28,56
85,40,79,80,92,20,34,77,28,56
85,40,79,80,92,20,34,77,28,56
```

**问题在于 Lua 的 `randomseed` 即使发生越界，也并不会报错，甚至连一个警告都没有**。所以需要我们自己小心的处理这个问题，以免留下隐患。比如可以像下面这样对其重写：

```lua
-- fixed the overflow of randomseed by checking on the Lua side
math.randomseed = function(seed)
    local bitsize = 32
    local lua_version = tonumber(_VERSION:match("%d%.*%d*"))

    local seed = math.floor(math.abs(seed))
    if seed >= (2^bitsize) then
        -- integer overflow, so reduce to prevent a bad seed
        seed = seed - math.floor(seed / 2^bitsize) * (2^bitsize)
    end
    if lua_version < 5.2 then
        -- 5.1 uses (incorrect) signed int
        math.randomseed(seed - 2^(bitsize-1))
    else
        -- 5.2 uses (correct) unsigned int
        math.randomseed(seed)
    end

    return seed
end
```

这里需要说明下，Lua 5.1 `randomseed` 用的其实是一个 signed int，并不是 unsigned int，这个问题在 5.2 中已经得到了修正。来源 [lua-wiki](http://lua-users.org/wiki/MathLibraryTutorial)：

> Another thing to be aware of is truncation of the seed provided. math.randomseed will call the underlying C function srand which takes an unsigned integer value. Lua will cast the value of the seed to this format. In case of an overflow the seed will actually become a bad seed, without warning [3] (note that Lua 5.1 actually casts to a signed int [4], which was corrected in 5.2).

最后，又一个好消息是：**LuaJIT 的 `randomseed` 是支持浮点数（double）的**。其并没有调用 `srand`，而是利用自己的 PRNG 算法实现的。赞一个 :-)

> The PRNG generates the same sequences from the same seeds on all platforms and makes use of all bits in the seed argument. math.random() without arguments generates 52 pseudo-random bits for every call. The result is uniformly distributed between 0.0 and 1.0. It's correctly scaled up and rounded for math.random(n [,m]) to preserve uniformity.
