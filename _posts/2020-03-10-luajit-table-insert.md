---
layout:     post
title:      "Lua table 如何实现最快的 insert?"
subtitle:   "Fastest Table Inserts"
date:       2020-03-10
author:     "ms2008"
header-img: "img/post-bg-2015.jpg"
catalog:    true
tags:
    - OpenResty
    - Lua
    - LuaJIT
typora-root-url: ..
---

前两天群里有个朋友贴出一段代码：

```lua
function _M.table_keys(self, tb)
    if type(tb) ~= "table" then
        ngx.log(ngx.WARN, type(tb) .. "was given to table_keys")
        return tb
    end

    local t = {}
    for key,_ in pairs(tb) do
        table.insert(t, key)
    end

    return t
end
```

用来拷贝 HTTP Header，是这样来调用的：

```lua
request.REQUEST_HEADERS_NAMES = twaf_func:table_keys(request_headers)
```

但是他发现，这个操作很耗性能。加了这句，就少了 "5000" 并发。

且不管他这 "5000" 并发是怎么计算出来的。今天，我们就来探讨下 table insert 最快的方法。

### CASE 1

```lua
local t = {}
local table_insert = table.insert

for i=1,1e7 do
    table_insert(t, i)
end
```

最经典的写法，LuaJIT 2.1 耗时：**1838ms**

> 题外话：根据 Lua Wiki 上的优化建议，local 化的变量会更快。但是这在 LuaJIT 上几乎已经没有了优势。

### CASE 2

根据 Lua Wiki 上的优化建议 [Lua 5.1 Optimization Notes][1]:

> Short inline expressions can be faster than function calls. `t[#t+1] = 0` is faster than `table.insert(t, 0)`.

```lua
local t = {}

for i=1,1e7 do
    t[#t+1] = i
end
```

优化后，LuaJIT 2.1 耗时：**1836ms**

似乎和 CASE-1 并没有明显差距，lua-resty-waf 的作者指出了[其中的问题][2]：

![](/img/in-post/loop-comp.png)

通过对比二者的 trace log，可以发现它们几乎没有明显区别，但是都调用了 `lj_tab_len` 来获取 `t` 的长度，这个操作的时间复杂度为 `O(log n)`，那么完成整个 insert 动作的时间复杂度就是 `O(n log n)`，是影响二者耗时的主要原因。

### CASE 3

我们尝试将 `lj_tab_len` 干掉，自己来计算 `t` 的长度。那么理论上完成整个 insert 动作的时间复杂度就简化为了 `O(n)`。

```lua
local t = {}
local c = 1

for i=1,1e7 do
    t[c] = i
    c = c + 1
end
```

结果 LuaJIT 2.1 耗时：**57ms**

一个简单的优化，性能就提升了惊人的 32 倍！还可以更快吗？

### 参考文献

- [Optimisation Coding Tips][1]
- [Fast(er)(est) Table Inserts in LuaJIT][2]
- [Performance of array creation in Lua][3]

[1]: http://lua-users.org/wiki/OptimisationCodingTips
[2]: https://www.cryptobells.com/fasterest-table-inserts-in-luajit
[3]: https://blog.jgc.org/2013/04/performance-of-array-creation-in-lua.html