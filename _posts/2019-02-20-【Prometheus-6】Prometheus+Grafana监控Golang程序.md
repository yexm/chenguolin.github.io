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
   
prometheus client 内部会自动生成以上几个监控 metrics 的源代码在 [go_collector.go](https://github.com/prometheus/client_golang/blob/master/prometheus/go_collector.go) 和 [process_collector.go](https://github.com/prometheus/client_golang/blob/master/prometheus/process_collector.go)

Golang 程序使用 Prometheus 统计监控指标示例代码如下
```
var (
    promCounterVecs   = make(map[string]*CounterVec)
    promGaugeVecs     = make(map[string]*GaugeVec)
    promHistogramVecs = make(map[string]*HistogramVec)
    promSummaryVecs   = make(map[string]*SummaryVecs)
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
![](https://github.com/chenguolin/chenguolin.github.io/blob/master/data/image/prometheus-monitor-golang-grafana-1.png?raw=true)
![](https://github.com/chenguolin/chenguolin.github.io/blob/master/data/image/prometheus-monitor-golang-grafana-2.png?raw=true)

Grafana dashbord 的JSON格式配置文件 [golang-process进程监控.json](https://github.com/chenguolin/chenguolin.github.io/blob/master/data/grafana/Golang-Process%E8%BF%9B%E7%A8%8B%E7%9B%91%E6%8E%A7.json)，使用Grafana Import即可恢复dashbord。

针对以上 Grafana 报表，具体的配置如下。注意，Prometheus 会自动添加 job 和 instance 两个label，其它label可以在 Prometheus server 采集配置文件里配置，或业务自行定义上报。

## ① Variables
1. datasource   (数据源字段)
   ```
   1). General
       * Type: Datasource
       * Label: Datasource
   2). Data source options
       * Type: Prometheus
       * Instance: [thanos](https://github.com/thanos-io/thanos)
    ```
2. cluster   (集群字段)
    ```
    1). General
       * Type: Query
       * Label: cluster
    2). Query Options
       * Data source: $datasource
       * Query: label_values(go_goroutines{job="golang-process-metrics"},cluster)    //cluster 字段可以在Prometheus Server采集配置文件里面配置
    ```
3. node   (节点字段)
    ```
    1). General
       * Type: Query
       * Label: node
    2). Query Options
       * Data source: $datasource
       * Query: label_values(go_goroutines{job="golang-process-metrics",cluster="$cluster"},instance)  //instance 是Prometheus默认自带的label字段
     ```
4. Interval  (一般情况下都需要设置Interval字段用来筛选数据)
    ```
    1). General
       * Type: Interval
       * Label: Interval
    2) Interval options
       * Values: 1m,2m,10m,2h
       * Auto Option: 开
       * Min Interval: 2m   //一般设置为Prometheus server抓取频率的2倍，如果是1m抓取一次metrics，则设置为2m
    ```
    
## ② Golang info
1. Go version  (Table)
   ```
   sql: count(go_info{job=~"golang-process-metrics",cluster=~"$cluster", instance=~"$node"}) by (cluster, instance, version)
   Instant: 开  //获取的是当前最新的时序数据
   ```
2. Process starttime  (Table)
   ```
   sql: sum(ceil(process_start_time_seconds{job=~"golang-process-metrics",cluster=~"$cluster", instance=~"$node"})*1000) by (cluster, instance)
   Instant: 开  //获取的是当前最新的时序数据
   ```
   
## ③ Process info
1. Process Goroutine  (Graph)
   ```
   sql: go_goroutines{job=~"golang-process-metrics",cluster=~"$cluster", instance=~"$node"}
   ```
2. Process thread    (Graph)
   ```
   sql: go_threads{job=~"golang-process-metrics",cluster=~"$cluster", instance=~"$node"}
   ```
3. Process open fds    (Graph)
   ```
   sql: process_open_fds{job=~"golang-process-metrics",cluster=~"$cluster", instance=~"$node"}  
   ```
4. Per second GC count    (Graph)
   ```
   sql: sum(rate(go_gc_duration_seconds_count{job=~"golang-process-metrics",cluster=~"$cluster", instance=~"$node"}[$Interval])) by (instance)
   ```
5. Average GC time   (Graph)
   ```
   sql: go_gc_duration_seconds_sum{job=~"golang-process-metrics",cluster=~"$cluster", instance=~"$node"} / go_gc_duration_seconds_count{job=~"golang-process-metrics",cluster=~"$cluster", instance=~"$node"}
   ```
6. Process Memory   (Graph)
   ```
   sql: process_resident_memory_bytes{job=~"golang-process-metrics",cluster=~"$cluster", instance=~"$node"}
   ```

## ③ Memory mallocs
1. Memstats   (Graph)
   ```
   sql A: go_memstats_alloc_bytes{job=~"golang-process-metrics",cluster=~"$cluster", instance=~"$node"}
   sql B: go_memstats_heap_inuse_bytes{job=~"golang-process-metrics",cluster=~"$cluster", instance=~"$node"}
   ```
2. Memstats gc sys bytes   (Graph)
   ```
   sql: go_memstats_gc_sys_bytes{job=~"golang-process-metrics",cluster=~"$cluster", instance=~"$node"}
   ```
3. Memstats buck hash sys bytes   (Graph)
   ```
   sql: go_memstats_buck_hash_sys_bytes{job=~"golang-process-metrics",cluster=~"$cluster", instance=~"$node"}
   ```
4. Memstats heap idle bytes   (Graph)
   ```
   sql: go_memstats_heap_idle_bytes{job=~"golang-process-metrics",cluster=~"$cluster", instance=~"$node"}
   ```
5. Memstats heap inuse bytes   (Graph)
   ```
   sql: go_memstats_heap_inuse_bytes{job=~"golang-process-metrics",cluster=~"$cluster", instance=~"$node"}
   ```
6. Memstats heap alloc bytes   (Graph)
   ```
   sql: go_memstats_heap_alloc_bytes{job=~"golang-process-metrics",cluster=~"$cluster", instance=~"$node"}
   ```

# 四. Prometheus告警
Prometheus 自带告警模块，允许用户自定义告警规则，可以发送到邮件、企业微信，还可以自定义接口发送到短信，非常方便。具体可以参考  https://prometheus.io/docs/alerting/overview/


