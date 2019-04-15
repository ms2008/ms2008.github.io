---
layout:     post
title:      "说下 Kong 的非主流 Service Mesh 之路"
subtitle:   "Service Mesh"
date:       2019-04-15
author:     "李飘柔"
header-img: "img/post-bg-kongtribuitor-shirt.png"
catalog:    true
tags:
    - Kong
    - Service Mesh
---

> **转载自** [聊一聊微服务网关 kong 近期的模型变迁][1]
><br/><br/>
> 我也不喜欢最新的 Kong Service 模型，这里讨论一些我曾经的疑问。

在 0.13.X 版本之前，Kong 的核心域模型名为 API Object Routes，从 0.13.X 版本开始，Kong 引入了 Service/Route Object，把 API Object 标记为 deprecated。在当前的 1.1.X 版本中关于 API Object 部分的配置已经被移除了。

乍看上去 Service/Route 模型就是把之前的 API Object 强行一分为二，在 [gitter][2] 讨论组里，Kong 的维护人员说这是为了做到一个关注分离，让 Route 与 Service 可以一对一甚至多对一的灵活配置。不过即使以 API 的方式，也可以配置多个 API 项，通过转发到同一个服务地址做到路由与服务的多对一。这样的理由显然不具说服力。

深入细节的海洋，这里有很多让人蛋疼的地方。我随便列举一些，算是概括使用这套新版 API 需要注意的地方：

- 需要先定义 Service，再定义 Route。因为定义 Route 的过程中需要 apply 到一个 Service 上面

- Service/Route 均不支持批量删除(bulk delete) 需要遍历列表的 API 然后根据 id 单独删除

- Service 如果上面定义了 Route 项，无法删除，必须先手动删除它的路由

- 列表 API 不会列出全部的路由，这里有个分页，需要用 offset 不停的递归到 `offset=null` 才算是理论上都遍历了一遍

- 通常来说，插件可以添加到一个 Service 或 Route 上面。但是要额外注意某些插件由于业务特性问题，只能添加到 Service，也有只能添加到 Route 的插件

到此为止，没有感受到这套新模型带来的任何好处，只是把以前明显更简洁利落直达要害的模型，变得臃肿复杂。**在反向代理视角下，之前的 API Object 是一个简洁利落的优秀建模。这种模式随着服务化发展会暴露出很多局限性。因为服务化的场景，关注点都是服务本身，作为基础设施的网关，将 API Object 一分为二，分离明确出 Service 这个概念，就很自然了。**

直到这里，Kong 进行模型变迁的目的渐渐明朗了。Kong 的功能设计在向 Service Mesh 靠拢。

<u>提起 Service Mesh，主流的声音还是 Istio/Conduit 这些实现。它们思路相似，通过 SideCar 的设计，让一个 native 语言编写的高性能代理服务(e.g. Envoy) 跟随容器启动，通过变更 iptables 规则把容器进程的网络包转发给代理进程。代理进程结合容器管理平台的 API，知晓服务的启停细节和服务发现配置，进行最终的网络调用</u>。

可以说 Kong 走了一条非主流的 Service Mesh 之路。以自己为中心，以 Service 模型为基本元素，结合插件系统来实现对微服务非侵入式的一系列加成 monitor、tracing、logging、health-check 而这些插件基本都要面向 Service 作为粒度（这里不能是 api-name，想象一下监控面板里充斥着 xx-api-oauth, xx-api-no-auth, xx-api-for-test xx-api-for-admin 这些其实 backend 指向同一个服务的混乱）。

其实，Kong 这个模式是有不少局限性的，比如说：

- Service 模型跟容器平台如何集成与保持同步，是否能做到主流容器管理平台向 Kong 自动注册

- 调用链下游的服务难以被这种模式 cover 到。通过插件很容易能拦截 ingress 流量。但是下游服务如果直连调用其他子服务，Kong 就没法感知了。除非在微服务环境的内部再来一个 Kong 当一个 RESTful 总线

- 协议支持: Kong 作为一个 L7 网关，只能代理 http 流量，遇到基于 tcp 的 rpc 就抓瞎了

当然这种模式也有它的好处，简单容易配置部署。只需要一套 Service/Route/Plugin 的配置文件初始化进去，简单的服务化体系就成型了。颇有四两拨千斤的意味。


[1]: https://zhuanlan.zhihu.com/p/40166203
[2]: https://gitter.im/Kong/kong
