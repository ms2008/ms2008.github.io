---
layout:     post
title:      Kong 插件加载机制源码解析
subtitle:   以请求的视角窥探 Kong 的数据走向
date:       2018-05-18
author:     ms2008
header-img: img/post-bg-kongtribuitor-shirt.png
catalog:    true
tags:
    - Kong
---

## 前言

我曾经在前面的文章中系统性的描述了下 Kong 的插件加载机制，这篇我将通过源码解析的方式呈现其数据走向。剔除掉第三方依赖，Kong 的核心代码结构如下：

```
kong/
├── api/
├── cluster_events/
├── cmd/
├── core/
├── dao/
├── plugins/
├── templates/
├── tools/
├── vendor/
│
├── cache.lua
├── cluster_events.lua
├── conf_loader.lua
├── constants.lua
├── init.lua
├── meta.lua
├── mlcache.lua
└── singletons.lua
```

执行 `kong start` 启动 Kong 之后，Kong 会将解析之后的配置文件保存在 `$prefix/.kong_env`，同时生成 `$prefix/nginx.conf`、`$prefix/nginx-kong.conf` 供 OpenResty 的使用。需要注意的是，<u>这三个配置文件在每次 Kong 启动后均会被覆盖，所以这里是不能修改自定义配置的</u>。如果需要自定义 OpenResty 配置，需要自己准备配置模板，然后启动的时候调用：`kong start -c kong.conf --nginx-conf custom_nginx.template` 即可。

在 `nginx-kong.conf` 里包含了 Kong 的 Lua 代码加载逻辑：

```perl
init_by_lua_block {
    kong = require 'kong'
    kong.init()
}
init_worker_by_lua_block {
    kong.init_worker()
}

upstream kong_upstream {
    server 0.0.0.1;
    balancer_by_lua_block {
        kong.balancer()
    }
    keepalive 60;
}

# ... 省略若干

location / {
    rewrite_by_lua_block {
        kong.rewrite()
    }
    access_by_lua_block {
        kong.access()
    }

    header_filter_by_lua_block {
        kong.header_filter()
    }
    body_filter_by_lua_block {
        kong.body_filter()
    }
    log_by_lua_block {
        kong.log()
    }
}
```

这里的 `kong` 就是 `kong/init.lua`，是 Kong 的入口模块，基本上是这样定义的：

```lua
local Kong = {}

function Kong.init()
  -- ... 省略若干
end

function Kong.init_worker()
  -- ... 省略若干
end

function Kong.rewrite()
  -- ... 省略若干
end

function Kong.access()
  -- ... 省略若干
end

function Kong.header_filter()
  -- ... 省略若干
end

function Kong.log()
  -- ... 省略若干
end

-- ... 省略若干
```

也就是在 OpenResty 的不同执行阶段，调用其不同的 `handler`。下面我将按照其不同执行阶段，逐个解析它的加载过程。


### 1. init

这个阶段主要的工作就是完成 Kong 的初始化。比如：数据层（`DAO`）的创建、路由的创建、插件的预加载等等：

```lua
-- retrieve kong_config
local conf_path = pl_path.join(ngx.config.prefix(), ".kong_env")
local config = assert(conf_loader(conf_path))

local dao = assert(DAOFactory.new(config)) instantiate long-lived DAO
local ok, err_t = dao:init()
if not ok then
  error(tostring(err_t))
end
```

鉴于这个阶段是 OpenResty 执行最早的阶段，其也要完成 `init.lua` 中顶层作用域中的 `require("kong.core.globalpatches")()` 功能。主要就是重写 `math.randomseed`，让它优先使用 OpenSSL 生成的种子，如果失败了再用 `ngx.now()*1000 + ngx.worker.pid()` 替代，主要是为了解决 PRNG 随机数的问题，背景知识可以参考这里[正确认识随机数][1]。另外还做了些小 hack，比如：将 `init_worker` 阶段中的 `sleep` 替换为阻塞式的 `sleep` 以解决这个阶段不能 yield 的问题。

插件的预加载也是一项重要的工作，像之后阶段运行的「phase 循环」均依赖这个时候装载完成的插件。需要注意的是，**这里预装载的插件是 Kong 所有已安装的插件（也包含自定义的插件），即使没有启用也会被装载**。装载插件的动作由 `load_plugins()` 这个函数完成，其中有一些关键步骤：

```lua
-- sort plugins by order of execution
table.sort(sorted_plugins, function(a, b)
  local priority_a = a.handler.PRIORITY or 0
  local priority_b = b.handler.PRIORITY or 0
  return priority_a > priority_b
end)

-- add reports plugin if not disabled
if kong_conf.anonymous_reports then
  local reports = require "kong.core.reports"

  local db_infos = dao:infos()
  reports.add_ping_value("database", kong_conf.database)
  reports.add_ping_value("database_version", db_infos.version)

  reports.toggle(true)

  sorted_plugins[#sorted_plugins+1] = {
    name = "reports",
    handler = reports,
    schema = {},
  }
end
```

会将插件按照优先级排序，之后阶段运行的「phase 循环」将会按照这个顺序来完成。需要注意的是，<u>Kong 默认会添加一个插件用来匿名发送使用统计，如果不想启用，可以在 Kong 配置文件中关闭这个选项</u>：

```ini
anonymous_reports = off
```

完成上述工作之后，Kong 会把他们缓存在 `singletons` 中。这个 `singletons` 可以理解为一个容器，用于承载之后各个 worker 中的需要模块，主要是为了解决不同模块之间的数据共享问题。

```lua
-- populate singletons
singletons.ip = ip.init(config)
singletons.dns = dns(config)
singletons.loaded_plugins = assert(load_plugins(config, dao))
singletons.dao = dao
singletons.configuration = config
```

值得一提的是，Kong 路由信息其实是缓存在 `core` 里的，也就是由 `core/handler.lua` 来提供。路由的加载由 `build_router()` 完成：

```lua
assert(core.build_router(dao, "init"))
```

这个函数原型接受两个参数：

- `DAO`

  数据层 `DAO` 对象，用于从数据库中查找 API 信息

- `version`

  版本号参数，缓存版本信息，主要是为了后续 worker 判断当前路由是否为最新版本。

在这个阶段话完成后，Kong 会将路由版本信息置为 `init`，可以通过下面这样来确认：

```sh
curl -i -s http://localhost:8001/cache/router:version
```


### 2. init_worker

这个阶段的第一个动作就是设置 worker 的随机种子，利用上个阶段被重写的 `math.randomseed()` 函数来完成。至于为什么不在上个阶段就完成种子的设置，是因为在 init 阶段还没有开始 fork worker，如果设置了种子，根据 fork 的 COW 特性会导致之后所有的 worker 的种子都是一样的。因此这个动作只能在这个阶段来完成。

接下来便是初始化 Kong 事件，Kong 的事件分为两部分：

- worker 之间的事件，由 `worker_events` 来处理
- cluster 节点之间的事件，由 `cluster_events` 来处理

```lua
-- init inter-worker events
local worker_events = require "resty.worker.events"

local ok, err = worker_events.configure {
  shm = "kong_process_events", -- defined by "lua_shared_dict"
  timeout = 5,            -- life time of event data in shm
  interval = 1,           -- poll interval (seconds)

  wait_interval = 0.010,  -- wait before retry fetching event data
  wait_max = 0.5,         -- max wait time before discarding event
}
if not ok then
  ngx_log(ngx_CRIT, "could not start inter-worker events: ", err)
  return
end

-- init cluster_events
local dao_factory   = singletons.dao
local configuration = singletons.configuration

local cluster_events, err = kong_cluster_events.new {
  dao                     = dao_factory,
  poll_interval           = configuration.db_update_frequency,
  poll_offset             = configuration.db_update_propagation,
}
if not cluster_events then
  ngx_log(ngx_CRIT, "could not create cluster_events: ", err)
  return
end
```

需要注意的是，从这个阶段开始需要执行一些 `hook`，定义在 `core/handler.lua` 大概是这个样子：

```lua
init_worker = {
  before = function()
    -- ... 省略若干
  end
},
rewrite = {
  before = function(ctx)
    ctx.KONG_REWRITE_START = get_now()
  end,
  after = function (ctx)
    ctx.KONG_REWRITE_TIME = get_now() - ctx.KONG_REWRITE_START -- time spent in Kong's rewrite_by_lua
  end
},
access = {
  before = function(ctx)
    -- ... 省略若干
  end
},
header_filter = {
  before = function(ctx)
    -- ... 省略若干
  end,
  after = function(ctx)
    -- ... 省略若干
  end
},

-- ... 省略若干
```

基本就是定义在各个阶段开始前、结束后执行的一些任务，多数是用来统计执行耗时。比如，这个阶段要执行的 `core.init_worker.before()` 就是用来注册上面的 worker 和 cluster 事件。

接下里就是初始化 Kong 缓存，并将路由版本信息的缓存置为 `init`。这里为什么不把路由信息缓存？很简单，无法解决序列化问题，所以只能缓存在 Lua Land 下。

```lua
local cache, err = kong_cache.new {
  cluster_events    = cluster_events,
  worker_events     = worker_events,
  propagation_delay = configuration.db_update_propagation,
  ttl               = configuration.db_cache_ttl,
  neg_ttl           = configuration.db_cache_ttl,
  resty_lock_opts   = {
    exptime = 10,
    timeout = 5,
  },
}
if not cache then
  ngx_log(ngx_CRIT, "could not create kong cache: ", err)
  return
end

local ok, err = cache:get("router:version", { ttl = 0 }, function()
  return "init"
end)
```

由于这个阶段只会执行一次，所以最后也要执行**所有预加载插件**的 `init_worker()` handler：

```lua
for _, plugin in ipairs(singletons.loaded_plugins) do
  plugin.handler:init_worker()
end
```

鉴于这个循环并没有完成任何插件生效策略的筛选，所以我没有把它归为「phase 循环」。


### 3. rewrite

这个阶段的逻辑比较简单，就是在开始前、和结束后分别执行两个 `hook` 将 Kong 处理耗时注入到 `ctx` 中：

```lua
local ctx = ngx.ctx
core.rewrite.before(ctx)

-- we're just using the iterator, as in this rewrite phase no consumer nor
-- api will have been identified, hence we'll just be executing the global
-- plugins
for plugin, plugin_conf in plugins_iterator(singletons.loaded_plugins, true) do
  plugin.handler:rewrite(plugin_conf)
end

core.rewrite.after(ctx)
```

中间完成的那个遍历，就是上面提到过多次的「phase 循环」，它的工作就是完成**插件生效策略的筛选**，并执行对应的 `handler`。不过这个阶段的筛选，只能筛选出启用的全局插件（因为路由匹配尚未开始，无法确定 API、Consumer）。

筛选的工作由 `plugins_iterator` 这个迭代器完成，定义在 `core/plugins_iterator.lua`，其函数原型是：

```lua
local function iter_plugins_for_req(loaded_plugins, access_or_cert_ctx)
  local ctx = ngx.ctx

  if not ctx.plugins_for_request then
    ctx.plugins_for_request = {}
  end

  local plugin_iter_state = {
    i                     = 0,
    ctx                   = ctx,
    api                   = ctx.api,
    loaded_plugins        = loaded_plugins,
    access_or_cert_ctx    = access_or_cert_ctx,
  }

  return setmetatable(plugin_iter_state, plugin_iter_mt)
end
```

接收两个参数：

- `loaded_plugins`

  前面预加载的所有插件，用于按优先级大小遍历插件。

- `access_or_cert_ctx`

  <u>根据参数命名也可以看出是标识当前阶段是否属于 access 或者是 ssl_certificate 阶段，以判断是否需要运行插件生效策略。不过 ssl_certificate 阶段未必都用，所以这里在 rewrite 阶段用来提前加载全局插件</u>。

其实这个函数真正起作用的是 `get_next` 这个 metamethod，是个递归函数：

```lua
local function get_next(self)
  -- load the plugin configuration in early phases
  if self.access_or_cert_ctx then
    -- ... 省略若干
    ctx.plugins_for_request[plugin.name] = plugin_configuration
  end

  -- return the plugin configuration
  local plugins_for_request = ctx.plugins_for_request
  if plugins_for_request[plugin.name] then
    return plugin, plugins_for_request[plugin.name]
  end

  return get_next(self) -- Load next plugin
end
```

可以看到这个阶段加载完成的全局插件会缓存在 `ctx.plugins_for_request`，这个对象用来保存**当前请求启用的插件**（包括下个阶段的局部插件）。

值得一提的是，纵观请求的整个处理过程，`plugins_iterator` 会被调用多次以完成整个插件生效策略的筛选。这个阶段的筛选只是第一个阶段，甚至都可以理解为不算筛选，而仅仅是为了加快全局插件的加载罢了。

> 事实上目前默认 Kong 自带的插件均没有需要这个阶段运行的 `handler`。


### 4. access

这个阶段就比较重要了，首先要执行的就是 `core.access.before(ctx)` 这个 `hook`，主要是完成路由的匹配。不过匹配前需要判断当前路由是否是最新版本，否则的话需要重建路由：

```lua
local version, err = singletons.cache:get("router:version", {
  ttl = 0
}, utils.uuid)
if err then
  log(ngx.CRIT, "could not ensure router is up to date: ", err)

elseif router_version ~= version then
  -- router needs to be rebuilt in this worker
  log(DEBUG, "rebuilding router")

  local ok, err = build_router(singletons.dao, version)
  if not ok then
    router_err = err
    log(ngx.CRIT, "could not rebuild router: ", err)
  end
end
```

接下来就是运行「phase 循环」了，来彻底完成插件生效策略的筛选（因为这个时候已经完成了路由查找，之后通过 API 可以找到 auth 插件，进而确定 Consumer，这也是为什么 auth 插件的优先级普遍比较高的原因）。插件的筛选还是由 `get_next` 来完成，关键代码如下：

```lua
if consumer then
  -- ... 省略若干
  if api then
    plugin_configuration = load_plugin_configuration(api.id, consumer_id, plugin.name)
  end
  if not plugin_configuration then
    plugin_configuration = load_plugin_configuration(nil, consumer_id, plugin.name)
  end
end

if not plugin_configuration then
  -- Search API specific, or global
  if api then
    plugin_configuration = load_plugin_configuration(api.id, nil, plugin.name)
  end
  if not plugin_configuration then
    plugin_configuration = load_plugin_configuration(nil, nil, plugin.name)
  end
end

ctx.plugins_for_request[plugin.name] = plugin_configuration
```

因此，最终的生效策略优先级就是：`api & consumer` > `consumer` > `api` > `global`

**需要注意的是，这个过程可能会覆盖上个阶段的全局插件，这正是插件生效策略的作用；同时该过程结束之后，当前请求需要启用的插件就已经最终确定，并被缓存在 `ctx.plugins_for_request` 中，直至该请求生命周期的结束。至于之后阶段运行的「phase 循环」其实就是直接读取的 `ctx.plugins_for_request` 而已。**

另外在这个阶段有个非常巧妙的设计，就是 `ctx.delay_response` 这个参数。它的原理就是把要执行的 `handler` wrap 在一个 `coroutine` 中，如果执行到一个插件需要 `ngx.say` 来提前执行 Nginx 的 content handler，那么它就会 `yield` 当前 `coroutine`，来延迟 content handler 的执行，并跳过之后需要执行的所有插件。这么做主要基于两点：

- 如果请求被插件拦截就尽快退出「phase」循环。
- 可以在输出 content 前，做一些自定义的操作。

```lua
ctx.delay_response = true

for plugin, plugin_conf in plugins_iterator(singletons.loaded_plugins, true) do
  if not ctx.delayed_response then
    local err = coroutine.wrap(plugin.handler.access)(plugin.handler, plugin_conf)
    if err then
      ctx.delay_response = false
      return responses.send_HTTP_INTERNAL_SERVER_ERROR(err)
    end
  end
end

if ctx.delayed_response then
  return responses.flush_delayed_response(ctx)
end
```

~~但是目前 Kong 在实现这块的时候也是有缺陷的，就是插件执行过程中如果 `ngx.say` 被触发，虽然将不会执行接下来的插件，但是依然在运行一个 hot 的迭代。这里其实完全可以避免，就像下面这样：~~

```lua
if not ctx.delayed_response then
  -- ... 省略若干
else
  break
end
```

> <span style="background-color: #FFFB00;">事实上 Kong 的做法完全没有问题，这么修改反而错了。因为全局插件在 rewrite 阶段已经加载完毕，在 access 阶段只需要加载完剩余的局部插件即可。在这个阶段即使插件产生了 circuiting，那么也应该遍历完所有插件（继续去加载优先级比较低的插件）。</span>
><br/><br/>
> 详情可以参考这里：[break the plugins iterator as soon as possible][3]

执行完插件的 `access()` handler 之后，就通过 `flush_delayed_response` 将延迟发送（如果需要的话）的 content 响应给客户端：

```lua
if ctx.delayed_response then
  return responses.flush_delayed_response(ctx)
end
```

如果通过了插件的 `access()` handler，却没有触发 content handler。那么接下来就是执行 `core.access.after` 这个 hook 了。其中的关键步骤:

```lua
local ok, err, errcode = balancer.execute(ctx.balancer_address)
if not ok then
  if errcode == 500 then
    err = "failed the initial dns/balancer resolve for '" ..
          ctx.balancer_address.host .. "' with: "         ..
          tostring(err)
  end
  return responses.send(errcode, err)
end
```

主要就是获取 `upstream`，并完成相关 DNS 的解析，其实这个事儿更应该是由 `balancer` 阶段来完成。只不过由于 `balancer` 不允许被 `yield`，因此就放在了这里。最后请求将会被 `proxy_pass` 到 kong_upstream，正式进入到 `balancer` 阶段。


### 5. balancer

> 这个阶段不会运行任何插件，当然也不会有「phase 循环」。

这个阶段的主要工作其实就是完成重试逻辑，因为在上个阶段的 `core.access.before` hook 已经完成了第一次节点的选取，这个阶段只是简单做了下 `ngx_balancer.set_current_peer` 和 `ngx_balancer.set_more_tries`。

重试依然使用的是 `balancer.execute`，关键步骤为:

```lua
if addr.try_count > 1 then
  -- only call balancer on retry, first one is done in `core.access.after` which runs
  -- in the ACCESS context and hence has less limitations than this BALANCER context
  -- where the retries are executed

  -- record failure data
  local previous_try = tries[addr.try_count - 1]
  previous_try.state, previous_try.code = get_last_failure()

  -- Report HTTP status for health checks
  if addr.balancer then
    if previous_try.state == "failed" then
      addr.balancer.report_tcp_failure(addr.ip, addr.port)
    else
      addr.balancer.report_http_status(addr.ip, addr.port, previous_try.code)
    end
  end

  local ok, err, errcode = balancer_execute(addr)
  if not ok then
    ngx_log(ngx_ERR, "failed to retry the dns/balancer resolver for ",
            tostring(addr.host), "' with: ", tostring(err))
    return ngx.exit(errcode)
  end

else
  -- first try, so set the max number of retries
  local retries = addr.retries
  if retries > 0 then
    set_more_tries(retries)
  end
end
```

值得一提的是 Kong 使用的 Ring-balancer 是自己实现的 [lua-resty-dns-client][2]，target 的选取默认使用的是 round-robin 算法，当 `upstream` 开启了 hash 时，则会换为一致性 hash。


### 6. header_filter

这个阶段将会接着执行「phase 循环」，只不过经过了 access 阶段之后，当前请求应该被执行的插件已经确定，并被缓存在自身中。这个阶段只是在遍历所有插件时将直接从上面的缓存中查找，并执行相应的 header_filter 方法，而不再经过生效策略的筛选，这当然也是出于性能上的考量。

```lua
local ctx = ngx.ctx
core.header_filter.before(ctx)

for plugin, plugin_conf in plugins_iterator(singletons.loaded_plugins) do
  plugin.handler:header_filter(plugin_conf)
end

core.header_filter.after(ctx)
```

最后执行的 `core.header_filter.after` hook，用于将 Kong 的处理时间注入到 header 中：

```lua
if singletons.configuration.latency_tokens then
  -- balancer 阶段执行时间
  header[constants.HEADERS.UPSTREAM_LATENCY] = ctx.KONG_WAITING_TIME
  -- access 阶段执行时间
  header[constants.HEADERS.PROXY_LATENCY]    = ctx.KONG_PROXY_LATENCY
end
```


### 7. body_filter

这个阶段其实和上个阶段差不多，只不过是在「phase 循环」中执行的 handler 为 `body_filter` 而已。这里就不再赘述了。

```lua
local ctx = ngx.ctx
for plugin, plugin_conf in plugins_iterator(singletons.loaded_plugins) do
  plugin.handler:body_filter(plugin_conf)
end

core.body_filter.after(ctx)
```


### 8. log

log 阶段依旧和上面的 filter 阶段差别不大：

```lua
local ctx = ngx.ctx
for plugin, plugin_conf in plugins_iterator(singletons.loaded_plugins) do
  plugin.handler:log(plugin_conf)
end

core.log.after(ctx)
```

不过是在 `core.log.after` hook 中需要更新 balancer 的被动健康检查状况而已。


***

## 结语

Kong 通过其插件扩展机制，提供了超越核心平台的额外功能和服务。<u>同时由于插件的启用是基于每请求的，会随着生命周期的结束而被销毁</u>。其被诟病的内存碎片问题，我猜测多少也和这一点的设计有些关系。


[1]: https://ms2008.github.io/2017/10/24/PRNG/
[2]: https://github.com/Kong/lua-resty-dns-client
[3]: https://github.com/Kong/kong/pull/4230