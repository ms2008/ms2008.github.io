---
layout:     post
title:      直观的表现 PRNG 周期性
subtitle:   无图无真相。真的真，真出声
date:       2017-12-13
author:     ms2008
header-img: img/post-bg-universe.jpg
catalog:    true
tags:
    - Lua-GD
    - Random
    - PRNG
typora-root-url: ..
---

我在前面的一些文章中介绍过：Lua 随机数算法用的是 LCG（32位的随机数，周期最多为2<sup>32</sup>）; LuaJIT 用的是 LFSR，周期达到 2<sup>223</sup>。下面是我分别用 Lua 和 LuaJIT 的随机数填充一个位图，代码：

```lua
#!/usr/bin/env lua
-- Draws the B/W image with lua-gd

local gd = require "gd"

local size = 512
local im = gd.createPalette(size, size)

local white = im:colorAllocate(255, 255, 255)
local black = im:colorAllocate(0, 0, 0)

math.randomseed(os.time())

for y = 1, size do
    for x = 1, size do
        local dot = math.random(2)
        if dot == 2 then
            im:setPixel(y, x, black)
        end
    end
end

im:png("lua-random.png")
```

- Lua

![](/img/in-post/lua-random.png)

- LuaJIT

![](/img/in-post/luajit-random.png)

可以看到，Lua 和 LuaJIT 几乎看不出区别。对于一般性的任务，一个设计良好的 LCG 算法可能已经足够「随机」了。

这个是知乎[@余天升]()用 php 的 `rand` 函数生成的随机数填充的位图:

![](/img/in-post/php-random.jpg)

哪个比较差一目了然。
