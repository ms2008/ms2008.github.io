---
layout:     post
title:      利用 loadstring 实现模块动态加载
subtitle:   Lua 中模块的动态加载
date:       2016-09-08
author:     ms2008
header-img: img/post-bg-js-module.jpg
catalog:    true
tags:
    - Lua
    - module
---

先来看一段 snippet：

```lua
do
    i = 32
    local i = 0
    f = loadstring("i = i + 1; print(i)")
    g = function () i = i + 1; print(i) end

    f() --> 33
    g() --> 1

    h = function () _G.i = _G.i + 1; print(_G.i) end
    h() --> 34
end
```

`f()` 输出33，`g()` 输出1。原因是第一个 i 是全局变量，第二个 i 是 `local` 变量，而同名的 `local` 变量总是覆盖掉全局变量。`loadstring` 产生的函数只能看到全局变量，因此 `f()` 输出33。如果想让 `g()` 函数访问全局变量 i，可以利用全局环境变量 `_G`，所以 `h()` 就返回了 34。

因此如果要使用 `loadstring` 从字符串动态加载 Lua 代码的话，也不应使用 Lua 全局变量来存放加载后的 Lua code chunk，而应该当作 Lua 模块来进行处理。比如下面这个例子：

```lua
local lua_src = [[
    local _M = {}
    local say = ngx.say

    function _M.run()
        say("hello world")
    end

    return _M
]]

local f, err = loadstring(lua_src, "module foo")

if not f then
    ngx.say("failed to load module: ", err)
    return
end

local mod = f()

if not mod then
    ngx.say("Lua source does not return the module")
    return
end

package.loaded.foo = mod
```

这相当于我们从内存（一个 Lua 字符串）直接加载了一个名为 `foo` 的 Lua 模块。然后每当要调用这片代码时，我们可以像使用模块一样：

```lua
local foo = require "foo"
foo.run()
```
