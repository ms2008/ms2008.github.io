---
layout:     post
title:      "也谈 ngx.ctx 继承问题"
subtitle:   "适合自己的才是最好的"
date:       2018-11-26
author:     "ms2008"
header-img: "img/post-bg-kongtribuitor-shirt.png"
catalog:    true
tags:
    - OpenResty
    - Kong
---

在前一阵子的 [OpenResty Con 2018][1] 上，来自又拍云的 [@tokers][2] 分享了他们对 `ngx.ctx` 的 hack，以确保在发生内部跳转后 `ngx.ctx` 的信息依旧不会丢失。其实这个 hack 早在去年就被 [@tokers][2] 分享到了社区：[ngx.ctx inheirt][3]，并且写了一篇文章来详细阐述其思路：[对 ngx.ctx 的一次 hack][4]

这回呢，[@tokers][2] 重新封装并开源了其实现：[lua-resty-ctxdump][5]。关于这个问题，其实 Kong 也遇到过，在 Kong 0.14 版本之前，当发生内部错误时（HTTP status 500），Kong 的 log 阶段插件就无法正常的被执行。无法运行的原因就是，Kong 会将状态码为 `4XX`、`5XX` 的请求内部跳转到另外一个 `location` 专门来处理这些异常状态：

```
error_page 400 404 408 411 412 413 414 417 /kong_error_handler;
error_page 500 502 503 504 /kong_error_handler;

location = /kong_error_handler {
    internal;
    uninitialized_variable_warn off;

    content_by_lua_block {
        kong.handle_error()
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

而 Kong 是将每个 API 启用的插件信息全部保存在 `ngx.ctx` 中的，所以一旦发生跳转，那么 Kong 自然就无法执行后续的插件了。Kong 针对这个问题也给了两种解决方案（[ISSUE-3193][6]）：

1. 去掉 `kong_error_handler` 禁止内部跳转

2. 就是利用 [@tokers][2] 恢复 `ngx.ctx` 的方案

最终 Kong 选择了第二种，不过 Kong 的实现和 [@tokers][2] 这回开源出来的 [lua-resty-ctxdump][5] 有些区别。就是 [lua-resty-ctxdump][5] 是将 `ngx.ctx` 引用在了自身的 `memo` table 中（Lua Land），也正因为如此，所以其提供的 `stash_ngx_ctx` 和 `apply_ngx_ctx` 方法**必须成对调用**，否则就会产生严重的内存泄漏。

```lua
local function ref_in_table(tb, key)
    if key == nil then
        return -1
    end

    local ref = tb[FREE_LIST_REF]
    if ref and ref ~= FREE_LIST_REF then
        tb[FREE_LIST_REF] = tb[ref]
    else
        ref = #tb + 1
    end

    tb[ref] = key

    return ref
end


function _M.stash_ngx_ctx()
    local ctx_ref = ref_in_table(memo, ngx.ctx)
    return ctx_ref
end


function _M.apply_ngx_ctx(ref)
    ref = tonumber(ref)
    if not ref or ref <= FREE_LIST_REF then
        return nil, "bad ref value"
    end

    local old_ngx_ctx = memo[ref]

    -- dereference
    memo[ref] = memo[FREE_LIST_REF]
    memo[FREE_LIST_REF] = ref

    return old_ngx_ctx
end
```

而 Kong 仅仅是将 `ngx.ctx` 的引用索引存放在了 `ngx.var` 中，随后根据这个索引把它从 Lua 的注册表中恢复出来。因为没有额外的索引创建动作，所以也就无需考虑引用释放问题。

```lua
function _M.stash_ref()
  local r = getfenv(0).__ngx_req
  if not r then
    ngx.log(ngx.WARN, "could not stash ngx.ctx ref: no request found")
    return
  end

  do
    local ctx_ref = ngx.var.ctx_ref
    if not ctx_ref or ctx_ref ~= "" then
      return
    end

    local _ = ngx.ctx -- load context if not previously loaded
  end

  local ctx_ref = C.ngx_http_lua_ffi_get_ctx_ref(r)
  if ctx_ref == FFI_NO_REQ_CTX then
    ngx.log(ngx.WARN, "could not stash ngx.ctx ref: no ctx found")
    return
  end

  ngx.var.ctx_ref = ctx_ref
end


function _M.apply_ref()
  local r = getfenv(0).__ngx_req
  if not r then
    ngx.log(ngx.WARN, "could not apply ngx.ctx: no request found")
    return
  end

  local ctx_ref = ngx.var.ctx_ref
  if not ctx_ref or ctx_ref == "" then
    return
  end

  ctx_ref = tonumber(ctx_ref)
  if not ctx_ref then
    return
  end

  local orig_ctx = registry.ngx_lua_ctx_tables[ctx_ref]
  if not orig_ctx then
    ngx.log(ngx.WARN, "could not apply ngx.ctx: no ctx found")
    return
  end

  ngx.ctx = orig_ctx
  ngx.var.ctx_ref = ""
end
```

如果从性能角度来考虑，我觉得 [lua-resty-ctxdump][5] 会更有优势，毕竟少了一些获取索引的动作。如果你的业务有大量的内部跳转，建议使用这个方案。同时需要注意的是 `stash_ngx_ctx` 和 `apply_ngx_ctx` 方法**必须成对调用**；如果你的业务和 Kong 类似，只是发生异常才会需要少量跳转，那么建议使用 Kong 的方案，`stash_ref` 和 `apply_ref` 方法也无需成对调用，也可以省点儿事儿。

[1]: http://con.openresty.org/cn/2018/
[2]: https://github.com/tokers
[3]: https://github.com/openresty/lua-nginx-module/issues/1057
[4]: https://segmentfault.com/a/1190000009485897
[5]: https://github.com/tokers/lua-resty-ctxdump
[6]: https://github.com/Kong/kong/issues/3193