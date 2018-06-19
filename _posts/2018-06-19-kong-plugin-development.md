---
layout:     post
title:      Kong 插件开发指南
subtitle:   Plugin Development
date:       2018-06-19
author:     ms2008
header-img: img/post-bg-kongtribuitor-shirt.png
catalog:    true
tags:
    - Kong
---

Kong 的插件使用了一个叫 [Classic][1] 的 class 机制。所有的插件都是从 `base_plugin.lua` 基类上继承而来。`base_plugin.lua` 定义了插件在各个阶段被执行的方法名：

```lua
function BasePlugin:init_worker()
  ngx_log(DEBUG, "executing plugin \"", self._name, "\": init_worker")
end

function BasePlugin:certificate()
  ngx_log(DEBUG, "executing plugin \"", self._name, "\": certificate")
end

function BasePlugin:rewrite()
  ngx_log(DEBUG, "executing plugin \"", self._name, "\": rewrite")
end

function BasePlugin:access()
  ngx_log(DEBUG, "executing plugin \"", self._name, "\": access")
end

function BasePlugin:header_filter()
  ngx_log(DEBUG, "executing plugin \"", self._name, "\": header_filter")
end

function BasePlugin:body_filter()
  ngx_log(DEBUG, "executing plugin \"", self._name, "\": body_filter")
end

function BasePlugin:log()
  ngx_log(DEBUG, "executing plugin \"", self._name, "\": log")
end
```

根据方法名也可以看出，这 7 个方法对应于 OpenResty 的不同执行阶段。也就是说插件只能对外暴露出这 7 个方法名中的一个或多个才能被 Kong 的插件机制执行，接下来 Kong 会在 OpenResty 不同的执行阶段，执行插件对应的方法。


### 1. 插件的结构

一个完整的插件目录结构应该像下面这样：

```
complete-plugin
├── api.lua
├── daos.lua
├── handler.lua
├── migrations
│   ├── cassandra.lua
│   └── postgres.lua
└── schema.lua
```

其各个模块的功能如下：

Module name      | Required | Description
:----------------|----------|------------
api.lua          | No       | 插件需要向 Admin API 暴露接口时使用
daos.lua         | No       | 数据层相关，当插件需要访问数据库时配置
handler.lua      | Yes      | 插件的主要逻辑，这个将会被 Kong 在不同阶段执行其对应的 handler
migrations/*.lua | No       | 插件依赖的数据表结构，启用了 daos.lua 时需要定义
schema.lua       | Yes      | 插件的配置参数定义，主要用于 Kong 参数验证

其中 `handler.lua` 和 `schema.lua` 是必需的，上面提到的插件需要暴露出来的方法就定义在 `handler.lua` 中。


### 2. 逻辑实现

这里以 Kong 自带的 request-termination 插件为例，分析其实现原理。其 `handler.lua` 的头部是这样定义的：

```lua
-- 引入插件基类
local BasePlugin = require "kong.plugins.base_plugin"
local responses = require "kong.tools.responses"

-- 派生出一个子类，其实这里是为了继承来自 Classic 的 __call 元方法，
-- 方便 Kong 在 init 阶段预加载插件的时候执行构造函数 new()
local RequestTerminationHandler = BasePlugin:extend()

-- 设置插件的优先级，Kong 将按照插件的优先级来确定其执行顺序（越大越优先）
-- 需要注意的是应用于 Consumer 的插件因为依赖于 Auth，所以 Auth 类插件优先级普遍比较高
RequestTerminationHandler.PRIORITY = 2
RequestTerminationHandler.VERSION = "0.1.0"
```

接下来就是定义插件的公有方法：

```lua
-- 插件的构造函数，用于初始化插件的 _name 属性，后面会根据这个属性打印插件名
-- 其实这个方法不是必须的，只是用于插件调试
function RequestTerminationHandler:new()
  RequestTerminationHandler.super.new(self, "request-termination")
end

-- 表明需要在 access 阶段执行此插件
function RequestTerminationHandler:access(conf)
  -- 执行父类的 access 方法，其实就是为了调试时输出日志用的
  RequestTerminationHandler.super.access(self)

  -- 接下来的就是插件的主要逻辑，不再赘述
  local status_code = conf.status_code
  local content_type = conf.content_type
  local body = conf.body
  local message = conf.message
  if body then
    ngx.status = status_code

    if not content_type then
      content_type = "application/json; charset=utf-8";
    end
    ngx.header["Content-Type"] = content_type
    ngx.header["Server"] = server_header

    ngx.say(body)

    return ngx.exit(status_code)
  else
    return responses.send(status_code, message)
  end
end

return RequestTerminationHandler
```


### 3. 参数定义

插件的参数定义在 `schema.lua` 中，类似于 [JSON Schema][2]，主要用于描述插件参数的数据格式：

```lua
local Errors = require "kong.dao.errors"

return {
  -- 描述插件参数的数据格式，用于 Kong 验证参数
  fields = {
    status_code = { type = "number", default = 503 },
    message = { type = "string" },
    content_type = { type = "string" },
    body = { type = "string" },
  },
  -- 自定义更为细粒度的参数校验
  self_check = function(schema, plugin_t, dao, is_updating)
    if plugin_t.status_code then
      if plugin_t.status_code < 100 or plugin_t.status_code > 599 then
        return false, Errors.schema("status_code must be between 100 .. 599")
      end
    end

    if plugin_t.message then
      if plugin_t.content_type or plugin_t.body then
        return false, Errors.schema("message cannot be used with content_type or body")
      end
    else
      if plugin_t.content_type and not plugin_t.body then
        return false, Errors.schema("content_type requires a body")
      end
    end

    return true
  end
}
```

当传入的参数无法通过 Kong 的校验时，插件配置将会失败，比如：

```sh
# 传入未定义的参数：foo
curl -s -i -X POST \
  --url http://localhost:8001/plugins/ \
  --data 'name=request-termination' \
  --data 'config.foo=bar'

HTTP/1.1 400 Bad Request
Date: Wed, 13 Jun 2018 15:03:39 GMT
Content-Type: application/json; charset=utf-8
Transfer-Encoding: chunked
Connection: keep-alive
Access-Control-Allow-Origin: *
Server: kong/0.12.3

{"config.foo":"foo is an unknown field"}

# 传入的 status_code 不是合法的数据
curl -s -i -X POST \
  --url http://localhost:8001/plugins/ \
  --data 'name=request-termination' \
  --data 'config.status_code=88'

HTTP/1.1 400 Bad Request
Date: Wed, 13 Jun 2018 15:05:12 GMT
Content-Type: application/json; charset=utf-8
Transfer-Encoding: chunked
Connection: keep-alive
Access-Control-Allow-Origin: *
Server: kong/0.12.3

{"config":"status_code must be between 100 .. 599"}
```

上面只是一个简单的应用，其实 Kong 还可以实现更为复杂的参数校验，具体可以参考这里：[Plugin Development - Store Configuration][3]

[1]: https://github.com/rxi/classic
[2]: http://json-schema.org/
[3]: https://getkong.org/docs/0.12.x/plugin-development/plugin-configuration/
