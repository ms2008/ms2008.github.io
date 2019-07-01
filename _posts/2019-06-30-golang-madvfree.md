---
layout:     post
title:      "Go 1.12 关于内存释放的一个改进"
subtitle:   "并不是内存泄露"
date:       2019-06-30
author:     "ms2008"
header-img: "img/post-bg-golang-blog-hero.jpg"
catalog:    true
tags:
    - Golang
typora-root-url: ..
---

一直以来 go 的 runtime 在释放内存返回到内核时，在 Linux 上使用的是 `MADV_DONTNEED`，虽然效率比较低，但是会让 RSS（resident set size 常驻内存集）数量下降得很快。不过在 go 1.12 里专门针对这个做了优化，runtime 在释放内存时，使用了更加高效的 `MADV_FREE` 而不是之前的 `MADV_DONTNEED`。具体可以参考这里：

- [runtime: use MADV_FREE on Linux if available][1]
- [runtime: use MADV_FREE on linux as well][2]

这样带来的好处是，一次 GC 后的内存分配延迟得以改善，runtime 也会更加积极地将释放的内存归还给操作系统，以应对大块内存分配无法重用已存在的堆空间的问题。不过也会带来一个副作用：**RSS 不会立刻下降，而是要等到系统有内存压力了，才会延迟下降**。需要注意的是，<u>`MADV_FREE ` 需要 4.5 以及以上内核，否则 runtime 会继续使用原先的 `MADV_DONTNEED` 方式</u>。

比如，我最近就遇到了这个问题：

![](/img/in-post/go1.12-madvfree.png)

几台服务的请求量差别并不大，可以明显看到 25（4.14 内核，其他几台都是 3.10 内核）的内存释放看起来很慢，但是 HEAP 占用显示却正常：

![](/img/in-post/go1.12-heap.png)

> 顺便说下，4.1 以上内核，最为引人注目的特性就是 eBPF，通过一个内核内置的字节码虚拟机，可以完成数据包过滤、调用栈跟踪、耗时统计、热点分析等等高级功能，是 Linux 系统的性能分析利器。

当然 go 1.12 为了避免像这样一些靠判断 RSS 大小的自动化测试因此出问题，也提供了一个 `GODEBUG=madvdontneed=1` 参数可以强制 runtime 继续使用 `MADV_DONTNEED`：[runtime: provide way to disable MADV_FREE][3]。但是显然正常情况下，我们都应该优先使用 `MADV_FREE`。

[1]: https://go-review.googlesource.com/c/go/+/135395/
[2]: https://github.com/golang/go/issues/23687
[3]: https://github.com/golang/go/issues/28466
