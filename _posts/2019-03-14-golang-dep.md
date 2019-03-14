---
layout:     post
title:      "Golang Dep 依赖冲突处理"
subtitle:   "Dep Dependencies"
date:       2019-03-14
author:     "ms2008"
header-img: "img/post-bg-golang-blog-hero.jpg"
catalog:    true
tags:
    - Golang
    - Dep
    - Prometheus
---

对于 Golang 应用内存堆栈的监控，基本都是读取 `runtime.MemStats`，然后发往一些 TSDB 进行可视化展示。代码一般都是这样的：

```go
memStats := &runtime.MemStats{}
runtime.ReadMemStats(memStats)
```

如果希望获取 GC 状态，可以这样：

```go
gcstats := &debug.GCStats{PauseQuantiles: make([]time.Duration, 100)}
debug.ReadGCStats(gcstats)
```

再简单点儿，可以直接用 Golang 的标准库 - [expvar][1]，使用也非常方便：

```go
package main

import (
	"expvar"
	"net/http"
)

func main() {
	http.Handle("/stats", expvar.Handler())
	http.ListenAndServe(":8080", nil)
}
```

之后直接访问 <http://localhost:8080/stats> ，就可以得到服务运行各项指标和状态：

```json
{
    "cmdline": [
        "/tmp/go-build076828032/b001/exe/main"
    ],
    "memstats": {
        "Alloc": 406936,
        "TotalAlloc": 406936,
        "Sys": 71629048,
        "Lookups": 0,
        "Mallocs": 1392,
        "Frees": 147,
        "HeapAlloc": 406936,
        "HeapSys": 66781184,
        "HeapIdle": 65683456,
        "HeapInuse": 1097728,
        "HeapReleased": 0,
        "HeapObjects": 1245,
        "StackInuse": 327680,
        "StackSys": 327680,
        "MSpanInuse": 16568,
        "MSpanSys": 32768,
        "MCacheInuse": 3456,
        "MCacheSys": 16384,
        "BuckHashSys": 1442888,
        "GCSys": 2234368,
        "OtherSys": 793776,
        "NextGC": 4473924,
        "LastGC": 0,
        "PauseTotalNs": 0,
        "PauseNs": [
            0,
            0
        ],
        "PauseEnd": [
            0,
            0
        ],
        "NumGC": 0,
        "NumForcedGC": 0,
        "GCCPUFraction": 0,
        "EnableGC": true,
        "DebugGC": false,
        "BySize": [
            {
                "Size": 0,
                "Mallocs": 0,
                "Frees": 0
            },
            {
                "Size": 8,
                "Mallocs": 47,
                "Frees": 0
            }
        ]
    }
}
```

当然 Prometheus 也内置了 Golang metrics 暴露的 handler，只需要简单调用即可实现，如下：

```go
package main

import (
	"net/http"

	"github.com/prometheus/client_golang/prometheus/promhttp"
)

func main() {
	http.Handle("/metrics", promhttp.Handler())
	http.ListenAndServe(":8080", nil)
}
```

访问 <http://localhost:8080/metrics> 即可，然后通过 Prometheus 聚合可以方便的在 Grafana 呈现应用运行的历史趋势。

然而通过这种方式，我在使用 dep 打包 vendor 的时候，遇到了个问题：

```
# .../vendor/github.com/prometheus/client_golang/prometheus
../vendor/github.com/prometheus/client_golang/prometheus/desc.go:88: undefined: model.IsValidMetricName
../vendor/github.com/prometheus/client_golang/prometheus/labels.go:56: model.LabelName(l).IsValid undefined (type model.LabelName has no field or method IsValid)
```

这个原因是由于 dep 在打包的时候用的是 Prometheus 的语义化版本 `0.9.2`，其依赖的 client_model proto 文件并不是最新的，具体可以参考[这里][2]。解决这个问题只需要覆盖掉 dep 的约束即可，手动编辑 Gopkg.toml 新增以下规则：

```
[[override]]
  name = "github.com/prometheus/client_golang"
  branch = "master"
```

再次执行 `dep ensure -update`，又出现：

```
unable to update checked out version: fatal: reference is not a tree: fa4aa9000d2863904891d193dea354d23f3d712a
```

至于这个是因为 dep 的缓存导致，清空掉 `$GOPATH/pkg/dep/` 即可。

[1]: http://docs.studygolang.com/pkg/expvar/
[2]: https://github.com/prometheus/client_golang/issues/353
