---
layout:     post
title:      Metatable 二三事
subtitle:   告别 Metatable 小白
date:       2016-10-12
author:     ms2008
header-img: img/post-bg-2015.jpg
catalog:    true
tags:
    - Lua
    - metatable
---

对于很多 Luaer 来说，也许只有 `table` 才会有 `metatable`。其实这种认识是错误的。

**实际上在 Lua 中，每个值都可以拥有一个元表。**对 `userdata` 和 `table` 类型而言，其每个值都可以拥有独立的元表，也可以几个值共享一个元表。对于其他类型，一个类型的值共享一个元表。例如所有 `number` 类型的值会共享一个元表。而默认 Lua 只为 `string` 类型赋予了元表，其他的都没有设置。以下代码就可以验证：

```lua
print(getmetatable(123))    --> nil
print(getmetatable(true))   --> nil
print(getmetatable({}))     --> nil
print(getmetatable("str"))  --> table: 0x***
```

使用 `getmetatable` 可以获取任意值的元表；而 `setmetatable` 仅仅可以设置 `table` 和 `userdata` 类型值的元表。如果想要设置其他类型的元表，只能通过 `debug` 库来修改。比如，以下是我定义的字符串相加的操作：

```lua
local str1 = "just for fun!"
local str2 = "pls go on"
local mt = getmetatable(str1)
mt.__add = function (s, d)
    return s .. d
end
debug.setmetatable(str1, mt)
print(str1+str2)

-- output:
-- just for fun!pls go on
```
