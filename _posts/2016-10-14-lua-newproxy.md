---
layout:     post
title:      使用 newproxy 生成 userdata
subtitle:   原生的 Lua 也是可以生成 userdata 的
date:       2016-10-14
author:     ms2008
header-img: img/post-bg-os-metro.jpg
catalog:    true
tags:
    - Lua
    - metatable
    - userdata
---

在 Lua 中，`userdata` 一般是在 C 里创建的数据结构，只能通过 C API 来操作。理论上 Lua 是不支持的，但是作者增加了一个隐藏的特性 `newproxy` 用于创建一个空的 `userdata`，参数可以选择是否带 `metatable`。主要是用来测试 GC，实现析构函数用的。

> newproxy is an unsupported and undocumented function in the Lua base library. From Lua code, the setmetatable function may only be used on objects of table type. The newproxy function circumvents that limitation by creating a zero-size userdata and setting either a new, empty metatable on it or using the metatable of another newproxy instance. We are then free to modify the metatable from Lua. This is the only way to create a proxy object from Lua which honors certain metamethods, such as __len.

我们倒是可以利用这个特性，来生成 `userdata` 给 `setmetatable` 使用。记得原先我们在 Lua 5.1 下测试 `table` 的 `__len` 是没有效果的，现在我们可以尝试下针对 `userdata` 的结果：

```lua
local t = newproxy(true)
getmetatable(t).__len = function()
    return 5
end
print(#t)

-- output:
-- 5
```

需要注意的是这个特性也是仅仅存在于 Lua 5.1 下，在 5.2 已经被移除了，因为从 5.2 开始，Lua 开始为 `table` 支持 `__len`、`__gc` 元方法了。
