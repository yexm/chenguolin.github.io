---
layout:  post  # 使用的布局（不需要改）
catalog: true  # 是否归档
author: 陈国林 # 作者
tags:          #标签
    - Prometheus
---

# 一. 背景
`Prometheus`做为CNCF第二名成员，事实上已经是云原生生态监控的标准。云原生生态体系绝大多数的开源组件都是使用Golang进行编写，例如Kubernetes项目。因此，使用`Prometheus`监控Golang程序也就成了开发必须要掌握的一个技能。`Prometheus`虽然是监控的标准，但在报表的展示上却需要`Grafana`配合，实际上`Prometheus + Grafana`的监控方案已经是业内公认的一个标准。

这篇文章会总结下如何使用 `Prometheus + Grafana` 监控Golang程序，包括实现细节、报表配置 以及报警配置。

# 二. 监控实现
Golang程序只需要引入并使用 [go-prometheus](https://github.com/chenguolin/go-prometheus) package，prometheus client 内部会自动生成以下几个服务本身监控 metrics，内部会自动起一个HTTP Server，端口默认使用`28888`

1. Goroutine监控
   + go_goroutines
   + go_threads
2. 内存分配监控
   + go_memstats_alloc_bytes
   + go_memstats_alloc_bytes_total
   + go_memstats_buck_hash_sys_bytes
   + go_memstats_frees_total
   + go_memstats_gc_cpu_fraction
   ...
3. GC监控
   + go_gc_duration_seconds
   + go_gc_duration_seconds_sum
   + go_gc_duration_seconds_count
4. Go信息
   + go_info
   
prometheus client 内部会自动生成以下几个服务本身监控 metrics 的源代码在 [go_collector.go](https://github.com/prometheus/client_golang/blob/master/prometheus/go_collector.go) 和 [process_collector.go](https://github.com/prometheus/client_golang/blob/master/prometheus/process_collector.go)

Golang 程序使用 Prometheus 统计监控指标示例代码如下
```
var (
    promCounterVecs   = make(map[string]*CounterVec)
    promGaugeVecs     = make(map[string]*GaugeVec)
    promHistogramVecs = make(map[string]*HistogramVec)
    promSummaryVecs = make(map[string]*SummaryVecs)
)

func init() {
    // init promCounterVecs
    promCounterVecs["one"] = NewCounterVec(
	"name",
	"desc",
	[]string{"label1", "label2" ...})
    ...

    // init promGaugeVecs
    promGaugeVecs["one"] = NewGaugeVec(
	"name",
	"desc",
	[]string{"label1", "label2" ...})
    ...

    // init promHistogramVecs
    promHistogramVecs["one"] = NewHistogramVec(
	"name",
	"desc",
	[]float64{60, 10 * 60, 30 * 60, 3600, 6 * 3600, 12 * 3600, 86400},
	[]string{"label1", "label2" ...})
    ...
    
    // init promSummaryVecs
    promSummaryVecs["one"] = NewSummaryVec(
	"name",
	"desc",
	[]float64{60, 10 * 60, 30 * 60, 3600, 6 * 3600, 12 * 3600, 86400},
	[]string{"label1", "label2" ...})
    ...
}

...
```

# 三. Grafana配置

# 四. Prometheus报警
