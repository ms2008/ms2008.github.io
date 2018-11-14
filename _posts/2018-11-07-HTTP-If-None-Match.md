---
layout:     post
title:      "If-None-Match 在刷票软件中的应用"
subtitle:   "那些刷票的骚操作"
date:       2018-11-07
author:     "ms2008"
header-img: "img/post-bg-universe.jpg"
catalog:    true
tags:
    - HTTP
typora-root-url: ..
---

**优化系统的极限就是不发送任何请求，这一点通常使用缓存来实现**。例如，在一些流量非常大 WEB 的系统中，我们通常会在源站前面启用 CDN。这样用户直接访问的是 CDN 中的缓存内容，降低真实服务端的压力。

![](/img/in-post/WEB-CDN.png)

同样服务端在输出响应时，可以通过响应头输出一些与缓存有关的信息，从而达到少发或不发请求的目的。

例如，服务端可以通过响应头里的 `Last-Modified`（最后修改时间） 或者 `ETag`（内容特征） 标记实体。浏览器会存下这些标记，并在下次请求时带上 `If-Modified-Since`: 上次 `Last-Modified` 的内容或 `If-None-Match`: 上次 `ETag` 的内容，询问服务端资源是否过期。如果服务端发现并没有过期，直接返回一个状态码为 304、正文为空的响应，告知浏览器使用本地缓存；如果资源有更新，服务端返回状态码 200、新的 `Last-Modified`、`Etag` 和正文。

<u>这样就解释了为什么我们在刷票的时候，明明看到有票，但是却无法下单（实际上已经没票了，你看到的只是缓存信息）</u>。所以如何绕过 CDN 拿到余票的最新信息，成为了抢票成功与否的关键。有一些刷票软件开辟了个新的思路：通过伪造 `If-None-Match` 头来跳过 CDN 缓存，尽快获取源站的最新数据。

`If-None-Match` 是一个条件式请求首部，对应校验的源站头部为 `ETag`，当且仅当服务器上没有任何资源的 `ETag` 属性值与这个首部中所列出的相匹配的时候，才会对请求进行相应的处理（有文件则响应200），如果匹配会直接给304（文件没有修改）。<u>如果源站也没有`ETag`这个头，这样 CDN 的缓存文件也没法校验这个头信息，当终端发起的请求中带这个头信息时，CDN 会将这样的请求回源去校验</u>。

分析完了原理，屏蔽这些刷票软件也变得非常简单：就是在 CDN 上配置策略，删掉 `If-None-Match`、`If-Modified-Since` 这些请求头，再进行后续的处理。实际上拦截效果也非常好：

![](/img/in-post/HTTP-If-None-Match.jpg)

***

### 参考文献

- [Nginx 配置之性能篇](https://imququ.com/post/my-nginx-conf-for-wpo.html)
