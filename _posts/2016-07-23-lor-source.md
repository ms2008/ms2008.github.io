---
layout:     post
title:      Lor 源码解析
subtitle:   ""
date:       2016-07-23
author:     ms2008
header-img: img/post-bg-re-vs-ng2.jpg
catalog:    true
tags:
    - Framework
    - OpenResty
    - Lua
    - Flame Graphs
typora-root-url: ..
---

lor 是一个基于 ngx_lua 的 MVC 框架，其 API 很类似于 Node 社区的著名框架 Express

lor 代码结构如下：

```
lor/
├── index.lua
├── lib
│   ├── application.lua
│   ├── debug.lua
│   ├── lor.lua
│   ├── methods.lua
│   ├── middleware
│   │   ├── cookie.lua
│   │   ├── init.lua
│   │   ├── params.lua
│   │   └── session.lua
│   ├── request.lua
│   ├── response.lua
│   ├── router
│   │   ├── layer.lua
│   │   ├── route.lua
│   │   └── router.lua
│   ├── utils
│   │   ├── path_to_regexp.lua
│   │   ├── path_to_regexp_lua.lua
│   │   └── utils.lua
│   ├── view.lua
│   └── wrap.lua
└── version.lua
```

#### 前言

在 web 开发中，一个简化的处理流程就是：客户端发起请求，然后服务端进行处理，最后返回相关数据。不管对于哪种语言哪种框架，除去细节的处理，简化后的模型都是一样的。客户端要发起请求，首先需要一个标识，通常情况下是 URL，通过这个标识将请求发送给服务端的某个具体处理程序，在这个过程中，请求可能会经历一系列全局处理，比如验证、授权、URL 解析等，然后定位到某个处理程序进行业务处理，最后将生成的数据返回客户端，客户端将数据结合视图模版呈现出合适的样式。

至于 url 是不是指向文件是无所谓的，但最终都是要根据其定位到某个具体的处理程序，也就是 url 到 handler 有个路由映射的过程，只不过不同的框架有不同的处理方法。

#### 1. app

使用 lor 创建一个简单的应用

```lua
local lor = require("lor.index")
local app = lor()

app:get("/", function(req, res, next)
    res:send("hello world!")
end)

app:run()
```

通过查看源码，可以发现 lor 其实是一个 wrap 对象，worker 内唯一。`lor()` 会调用自身的元方法来创建一个 app 实例（每请求）。相关源码如下：

```lua
local Application = require("lor.lib.application")
local Wrap = require("lor.lib.wrap")

LOR_FRAMEWORK_DEBUG = false

local createApplication = function(options)
    if options and options.debug and type(options.debug) == 'boolean' then
        LOR_FRAMEWORK_DEBUG = options.debug
    end

    local app = Application:new()
    app:init(options)

    return app
end


local lor = Wrap:new(createApplication, Router, Route, Request, Response)
return lor
```

可以看出，事实上 lor 是将请求交给了 app，通过 `app:run()` 来处理了（其实最终是 `app:handle()`）。

对于 lor 每个请求都会生成一个 app 实例，一个 app 实例有且仅有一个 router 对象，在这个对象里有一个 stack 数组，里面保存了所有 layer（中间件） 相关的信息

> **注意:** 其实未必是一个 Router， 这里指的是一个主 Router ，其下面可能会挂载很多子 router（路由组）和普通的 route

#### 2. 中间件

一个 lor 的应用从本质来说就是一系列的中间件的调用。其实，一个中间件，就是一个函数。通常情况下，一个中间件函数（通常都是匿名函数）的形式如下：
```lua
function (req, res, next) {
	-- do something
}
```

如果是错误处理的中间件，形式如下：
```lua
function (err, req, res, next) {
	-- do something
}
```

参数中，req 和 res 分别表示请求的 request 和 response ，next 本身也是一个函数，调用 `next()` 就会继续执行下一个中间件。其实，请求的处理过程就是依次经过各个中间件。

中间件大体上可以分为两种：普通中间件和路由中间件。注册普通中间件，通常是通过 `app.use()` 方法；而注册路由中间件，通常是通过 `app.METHOD()` 方法。例如：

```lua
app:use("/user", function(req, res, next)
    count = 1
    next()
end)

app:get("/user/123", function(req, res, next)
    count = 2
end)
```

以上两者主要区别在于：
* 前者匹配所有以 /user 开始的路径，而后者会精确匹配 /user/123 路径；
* 前者对于请求的方法没有限制，而后者只能处理方法为GET的请求。

在了解请求处理的详细过程之前，需要先来了解 Router。

#### 3. router

简单来说，Router 就是一个中间件的容器。事实上，Router 是 lor 一个非常核心的东西。App 的很多 API，例如：app:use, app:handle 等，都是对 Router 的一个简单封装。

Router 对象有一个 stack 属性，是一个数组，里面存放了所有的中间件。当调用 `app:use` 来注册中间件的时候，实际上是执行了 `router:use`，从而向 router.stack 数组中添加中间件。router.stack 中的每一项是一个 layer 对象，它是对中间件函数的一个封装。添加中间件的部分相关源码如下：

```lua
function Router:use(path, fn, fn_args_length)
    local layer
    if type(fn) == "function" then -- fn is a function
        layer = Layer:new(path, {
            is_end = false,
            is_start = true
        }, fn, fn_args_length)
    else -- fn is a group router
        layer = Layer:new(path, {
            is_end = false,
            is_start = true
        }, fn.call(fn), fn_args_length)

        local group_router_stack = fn.stack
        if group_router_stack and not fn.is_repatterned then
            fn.is_repatterned = true -- fixbug: fn.is_repatternd to remember, avoid 404 error when "lua_code_cache on"
            for i, v in ipairs(group_router_stack) do
                v.pattern = utils.clear_slash("^/" .. path .. v.pattern)
            end
        end

        debug("router.lua#use-inner now the group router(" .. fn.name .. ") stack is:")
        debug(function()
            for i, v in ipairs(fn.stack) do
                print(i, v)
            end
        end)
        debug("router.lua#use-inner now the group router(" .. fn.name .. ") stack is------\n")
    end

    tinsert(self.stack, layer)

    debug("router.lua#use now the router(" .. self.name .. ") stack is:")
    debug(function()
        for i, v in ipairs(self.stack) do
            print(i, v)
        end
    end)
    debug("router.lua#use now the router(" .. self.name .. ") stack is------\n")

    return self
end
```

上面是添加普通的中间件和路由组，大概如此。不过，对于路由中间件，就稍微复杂了些。在此之前，先看下添加路由中间件的方法：

```lua
-- app:get -> router:app_route -> route:get
app:get("/index", function(req, res, next)
    res:send("hello world!")
end)

-- application.lua
function App:init_method()
    for http_method, _ in pairs(supported_http_methods) do
        self[http_method] = function(self, path, fn)
            debug("\napp:" .. http_method, path, "start init##############################")
            local route = self.router:app_route(path)
            route[http_method](route, fn) -- like route:get(fn)
            debug("app:" .. http_method, path, "end init################################\n")
            return self
        end
    end
end
```

可以看到，添加路由中间件是通过 `router.app_route()` 来创建一条新的路由，然后调用 `route:http_method(fn)` 来注册相关的处理函数。

Route 可以简单理解为存放路由处理函数的容器，它也有一个 stack 属性，为一个数组，其中的每一项也是一个 layer 对象，是对路由处理函数的包装。下面来看当执行 `router.app_route()` 的时候发生了什么：

```lua
function Router:app_route(path)
    local route = Route:new(path)
    local layer = Layer:new(path, {
        is_end = true,
        is_start = true
    }, route, 3) -- important: a magick to supply route:dispatch
    layer.route = route

    tinsert(self.stack, layer)

    debug("router.lua#route now the router(" .. self.name .. ") stack is:")
    debug(function()
        for i, v in ipairs(self.stack) do
            print(i, v)
        end
    end)
    debug("router.lua#route now the router(" .. self.name .. ") stack is++++++\n")

    return route
end
```

也就是说，当调用 `router.app_route()` 的时候，实际上是新建了一个 layer 放在 router.stack 中，并设置 layer.route 为新建的 route 对象，最后把 route 对象 return 出来。

下面来看 `route:http_method(fn)` 的时候发生了什么：
```lua
function Route:initMethod()
    for http_method, _ in pairs(supported_http_methods) do
        self[http_method] = function(self, fn)
            local layer = Layer:new("/", {
                is_end = true
            }, fn, 3)
            layer.method = http_method
            self.methods[http_method] = true
            tinsert(self.stack, layer)

            debug("route.lua# now the route(" ..  self.name .. ") stack is:")
            debug(function()
                for i, v in ipairs(self.stack) do
                    print(i, v)
                end
            end)
            debug("route.lua# now the route(" ..  self.name .. ") stack is~~~~~~~~~~~~\n")
        end
    end
end
```
即：当调用 `route:http_method(fn)` 的时候，新建了一个 layer 放在了 route.stack 中。通过上面分析发现，router 其实是一个二维的结构。

#### 4. 请求处理过程

> 前面所提到的，无论是添加中间件，还是注册路由，都是应用的构建过程。当应用构建好了之后，客户端发起请求，这个时候，应用就开始使用前面的中间件和参数处理函数，来处理客户端的请求。

前面提到，所有的请求，都是由 `app.handle()` 来处理的，通过看源码，可以发现，其实 `app.handle()` 是调用了 `router.handle()`。

当请求到来时，经过中间件的顺序大致如下所示：

![Alt text](/img/in-post/lor-handler-1.png)

router.stack 中存的是一个个的 layer 对象，用来管理中间件。如果 layer 对象表示的是一个路由中间件，则其 route 属性会指向一个 route 对象，而 route.stack 中存放的也是一个个的 layer 对象，用来管理路由处理函数。

因此，当一个请求到来的时候，会依次通过 router.stack 中的 layer 对象，如果遇到路由中间件，则会依次通过 route.stack 中的 layer 对象。

对于 router.stack 中的每个 layer 对象：
* 会先判断是否匹配请求路径，如果不匹配，则跳过，继续下一个。
* 在路径匹配的情况下，如果是非路由中间件，则执行该中间件函数；
* 如果是路由中间件，则继续判断该中间件的路由对象能够处理请求的 HTTP 方法，如果不能够处理，则跳过继续下一个，如果能够处理则对 route.stack 中的 layer 对象（与请求的 HTTP 方法匹配的）依次执行。示例图如下（路由组）：

![Alt text](/img/in-post/lor-handler-2.png)

#### 5. lor 不足

* `app` 的构建过程不够 `lazy`，也就是每次请求进来都需要不断的构建 `layer`，即使是不匹配的 `path`
* 错误处理插件需要不断遍历 `layer` ，虽然不用执行 `fn`

> **提升性能的一个小** `trick` :

**Before**:
```lua
-- main.lua
local lor = require("lor.index")
local app = lor()

app:get("/", function(req, res, next)
    res:send("hello world!")
end)

app:run()
```

**After**:
```lua
-- main.lua
local app = require("app.app")
app:run()

-- app.lua
local lor = require("lor.index")
local app = lor()

app:get("/", function(req, res, next)
    res:send("hello world!")
end)

return app
```

**性能比较**:
![Alt text](/img/in-post/lor-flame.png)


----
##### 参考资料

* [Lor Framework API Document](http://lor.sumory.com/apis/v0.1.0/)
* [深入理解 Express](http://syaning.com/2015/05/20/dive-into-express/)
