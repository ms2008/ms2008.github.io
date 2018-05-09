---
layout:     post
title:      require 理解
subtitle:   Lua require 模块加载机制
date:       2016-09-07
author:     ms2008
header-img: img/home-bg-o.jpg
catalog:    true
tags:
    - Lua
    - module
---

在 lua 中加载的其他文件的代码，通常可以使用 `dofile`、`loadfile`、`require` 函数等来完成。其中 `dofile` 每次加载都要编译执行，效率比较低，所以不推荐使用；同样 `loadfile` 虽然只需编译一次，但是并没有把结果缓存到 lua vm 中；因而，我们这里总是推荐使用第三种方式 `require`。

`require` 能够避免多次重复加载模块，一个模块被加载后会被缓存到 `pacakge.loaded`。 如果需要重新加载模块，可以清理 `package.loaded.test = nil`。

需要注意的是，require() 函数并没有使用全局变量，它是在 `package.loaded` 表里缓存已经加载的 Lua module 的。而 `package.loaded` 是挂载在 Lua VM 的 `registry` 表里的，不同于全局变量的环境表。

另外：

1. Lua module 不一定是 `table`，也可以是 `function`，只是 `table` 比较常见罢了。`function` 的一个例子是 LuaJIT 2.1 的 `require("table.new")`.
2. 在 Lua module 文件里的顶层作用域里声明的 `local` 变量一般会通过 upvalue 的形式挂载到 Lua module 里使用到这些变量的 `function` 里，而这些 `function` 一般会注册到 Lua module 的 `table` 里面，或者进一步以 `upvalue` 的形式挂载到这样的其他 `function` 中

所以，**lua module 的顶层作用域是不可以用来声明需要变动的值的**。并且，在 Lua 里加载模块的正确方式是：

```lua
local foo = require "foo"
```

直接写：

```lua
require "foo"
```

其实是错误的，因为它依赖于 `module()` 函数创建和模块同名的 Lua 全局变量的副作用。而使用 `module()` 函数在 Lua 社区里也是不推荐的（以至于 Lua 5.2 语言里干脆移除了 `module` 这个内建函数）。

最后，lua 的 vm 和 luajit 的 vm 有些行为是不同的，这一点在 or 的 github 就有提到：

> As the standard Lua 5.1 interpreter's VM is not fully resumable, the methods ngx.location.capture, ngx.location.capture_multi, ngx.redirect, ngx.exec, and ngx.exit cannot be used within the context of a Lua pcall() or xpcall() or even the first line of the for ... in ... statement when the standard Lua 5.1 interpreter is used and the attempt to yield across metamethod/C-call boundary error will be produced. Please use LuaJIT 2.x, which supports a fully resumable VM, to avoid this.
