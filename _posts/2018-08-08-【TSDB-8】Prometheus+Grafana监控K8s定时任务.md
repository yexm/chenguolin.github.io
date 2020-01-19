---
layout:  post  # 使用的布局（不需要改）
catalog: true  # 是否归档
author: 陈国林 # 作者
tags:          #标签
    - TSDB
---

# 一. 背景
`Prometheus`做为CNCF第二名成员，事实上已经是云原生生态监控的标准。云原生生态体系绝大多数的开源组件都是使用Golang进行编写，例如Kubernetes项目。`Prometheus`虽然是监控的标准，但在报表的展示上却需要`Grafana`配合，因此 `Prometheus + Grafana`的监控方案已经是业内公认的一个标准。

这篇文章会总结下如何使用 `Prometheus + Grafana` 监控 K8s 定时任务，包括实现细节、报表配置 以及报警配置。

# 二. 监控实现
Golang 程序需要引入并使用 [go-prometheus](https://github.com/chenguolin/go-prometheus) package，prometheus client 内部会自动生成以下几个服务本身监控 metrics，内部会自动起一个HTTP Server，端口默认使用 `28888`，然后在 Prometheus server 端内部配置采集对应的服务地址，Prometheus 会定时来 pull 数据。

对于独立的监控指标，在Golang程序内部我们一般会放在 `pkg/prometheus/metrics.go` 内，如下所示

```
var (
    promCounterVecs   = make(map[string]*CounterVec)
    promGaugeVecs     = make(map[string]*GaugeVec)
    promHistogramVecs = make(map[string]*HistogramVec)
)

func init() {
    // k8s cronJob
    promGaugeVecs["k8s_batch_total_cronjob_running"] = NewGaugeVec(
        "k8s_batch_total_cronjob_running",
	"k8s batch total running cronjob",
	[]string{"cluster", "namespace", "cronjob"})
    promGaugeVecs["k8s_batch_total_cronjob_suspend"] = NewGaugeVec(
	"k8s_batch_total_cronjob_suspend",
	"k8s batch total suspend cronjob",
	[]string{"cluster", "namespace", "cronjob"})
    promHistogramVecs["k8s_batch_running_cronjob_starttime"] = NewHistogramVec(
	"k8s_batch_running_cronjob_starttime",
	"k8s batch running cronjob start time second",
	[]float64{3600, 86400, 3 * 86400, 7 * 86400, 15 * 86400, 30 * 86400},
	[]string{"cluster", "namespace", "cronjob"})

    // k8s job
    promGaugeVecs["k8s_batch_total_job_completed"] = NewGaugeVec(
	"k8s_batch_total_job_completed",
	"k8s batch total completed job",
	[]string{"cluster", "namespace", "cronjob"})
    promGaugeVecs["k8s_batch_total_job_running"] = NewGaugeVec(
	"k8s_batch_total_job_running",
	"k8s batch total running job",
	[]string{"cluster", "namespace", "cronjob"})
    promGaugeVecs["k8s_batch_total_job_failed"] = NewGaugeVec(
	"k8s_batch_total_job_failed",
	"k8s batch total failed job",
	[]string{"cluster", "namespace", "cronjob"})

    promGaugeVecs["k8s_batch_running_job_req_cpu"] = NewGaugeVec(
	"k8s_batch_running_job_req_cpu",
	"k8s batch running job request cpu",
	[]string{"cluster", "namespace", "cronjob", "job"})
    promGaugeVecs["k8s_batch_running_job_limit_cpu"] = NewGaugeVec(
	"k8s_batch_running_job_limit_cpu",
	"k8s batch running job limit cpu",
	[]string{"cluster", "namespace", "cronjob", "job"})
    promGaugeVecs["k8s_batch_running_job_req_mem"] = NewGaugeVec(
	"k8s_batch_running_job_req_mem",
	"k8s batch running job request mem",
	[]string{"cluster", "namespace", "cronjob", "job"})
    promGaugeVecs["k8s_batch_running_job_limit_mem"] = NewGaugeVec(
	"k8s_batch_running_job_limit_mem",
	"k8s batch running job limit mem",
	[]string{"cluster", "namespace", "cronjob", "job"})

    promHistogramVecs["k8s_batch_running_job_starttime"] = NewHistogramVec(
	"k8s_batch_running_job_starttime",
	"k8s batch running job start time second",
	[]float64{60, 10 * 60, 30 * 60, 3600, 6 * 3600, 12 * 3600, 86400},
	[]string{"cluster", "namespace", "cronjob", "job"})
}

// ResetPromMetrics prometheus metrics
func ResetPromMetrics() {
    promGaugeVecs["k8s_batch_total_cronjob_suspend"].Reset()
    promGaugeVecs["k8s_batch_total_cronjob_running"].Reset()
    promHistogramVecs["k8s_batch_running_cronjob_starttime"].Reset()
    promGaugeVecs["k8s_batch_total_job_completed"].Reset()
    promGaugeVecs["k8s_batch_total_job_running"].Reset()
    promGaugeVecs["k8s_batch_total_job_failed"].Reset()
    promGaugeVecs["k8s_batch_running_job_req_cpu"].Reset()
    promGaugeVecs["k8s_batch_running_job_limit_cpu"].Reset()
    promGaugeVecs["k8s_batch_running_job_req_mem"].Reset()
    promGaugeVecs["k8s_batch_running_job_limit_mem"].Reset()
    promHistogramVecs["k8s_batch_running_job_starttime"].Reset()
}

// SetK8sCronJobPromMetrics add k8s cronJob prometheus metrics
func SetK8sCronJobPromMetrics(cluster, ns, cronJob string, suspend bool) {
    promLabels := make(Labels)
    promLabels["cluster"] = cluster
    promLabels["namespace"] = ns
    promLabels["cronjob"] = cronJob

    if suspend {
        promGaugeVecs["k8s_batch_total_cronjob_suspend"].Set(promLabels, 1)
    } else {
	promGaugeVecs["k8s_batch_total_cronjob_running"].Set(promLabels, 1)
    }
}

// SetK8sRunningCronJobStartTimePromMetrics add k8s running cronJob starttime prometheus metrics
func SetK8sRunningCronJobStartTimePromMetrics(cluster, ns, cronJob string, starttime int64) {
    promLabels := make(Labels)
    promLabels["cluster"] = cluster
    promLabels["namespace"] = ns
    promLabels["cronjob"] = cronJob
    promHistogramVecs["k8s_batch_running_cronjob_starttime"].Observe(promLabels, float64(time.Now().Unix()-starttime))
}

// SetK8sJobPromMetrics add k8s Job prometheus metrics
func SetK8sJobPromMetrics(cluster, ns, cronJob string, status types.JobRunningStatus, cnt int) {
    promLabels := make(Labels)
    promLabels["cluster"] = cluster
    promLabels["namespace"] = ns
    promLabels["cronjob"] = cronJob

    if status == types.SuccessfulFinished {
	promGaugeVecs["k8s_batch_total_job_completed"].Set(promLabels, float64(cnt))
    } else if status == types.Running {
	promGaugeVecs["k8s_batch_total_job_running"].Set(promLabels, float64(cnt))
    } else {
        promGaugeVecs["k8s_batch_total_job_failed"].Set(promLabels, float64(cnt))
    }
}

// SetK8sRunningJobStartTimePromMetrics add k8s running job starttime prometheus metrics
func SetK8sRunningJobStartTimePromMetrics(cluster, ns, cronJob, job string, starttime int64) {
    promLabels := make(Labels)
    promLabels["cluster"] = cluster
    promLabels["namespace"] = ns
    promLabels["cronjob"] = cronJob
    promLabels["job"] = job
    promHistogramVecs["k8s_batch_running_job_starttime"].Observe(promLabels, float64(time.Now().Unix()-starttime))
}

// SetK8sRunningJobReqCPUPromMetrics add k8s running job request cpu prometheus metrics
func SetK8sRunningJobReqCPUPromMetrics(cluster, ns, cronJob, job string, cpu int64) {
    promLabels := make(Labels)
    promLabels["cluster"] = cluster
    promLabels["namespace"] = ns
    promLabels["cronjob"] = cronJob
    promLabels["job"] = job
    promGaugeVecs["k8s_batch_running_job_req_cpu"].Set(promLabels, float64(cpu))
}

// SetK8sRunningJobLimitCPUPromMetrics add k8s running job limit cpu prometheus metrics
func SetK8sRunningJobLimitCPUPromMetrics(cluster, ns, cronJob, job string, cpu int64) {
    promLabels := make(Labels)
    promLabels["cluster"] = cluster
    promLabels["namespace"] = ns
    promLabels["cronjob"] = cronJob
    promLabels["job"] = job
    promGaugeVecs["k8s_batch_running_job_limit_cpu"].Set(promLabels, float64(cpu))
}

// SetK8sRunningJobReqMemPromMetrics add k8s running job request memory prometheus metrics
func SetK8sRunningJobReqMemPromMetrics(cluster, ns, cronJob, job string, mem int64) {
    promLabels := make(Labels)
    promLabels["cluster"] = cluster
    promLabels["namespace"] = ns
    promLabels["cronjob"] = cronJob
    promLabels["job"] = job
    promGaugeVecs["k8s_batch_running_job_req_mem"].Set(promLabels, float64(mem))
}

// SetK8sRunningJobLimitMemPromMetrics add k8s running job limit memory prometheus metrics
func SetK8sRunningJobLimitMemPromMetrics(cluster, ns, cronJob, job string, mem int64) {
    promLabels := make(Labels)
    promLabels["cluster"] = cluster
    promLabels["namespace"] = ns
    promLabels["cronjob"] = cronJob
    promLabels["job"] = job
    promGaugeVecs["k8s_batch_running_job_limit_mem"].Set(promLabels, float64(mem))
}
```

注意: 为什么代码里面会有一个 ResetPromMetrics 函数，主要的原因是 `因为我们定时的metrics的label值会一直变，如果没有及时reset对应的metrcs的话，会导致内存的Prometheus metrics指标无限膨胀，每次Prometheus server来pull数据的时候都会拉取到很多无用的数据，干扰了我们的监控，因此我们需要在每次set metrics之前做些reset。`

# 三. Grafana配置

![](https://github.com/chenguolin/chenguolin.github.io/blob/master/data/image/grafana-k8s-cronjob-1.png?raw=true)
![](https://github.com/chenguolin/chenguolin.github.io/blob/master/data/image/grafana-k8s-cronjob-2.png?raw=true)
![](https://github.com/chenguolin/chenguolin.github.io/blob/master/data/image/grafana-k8s-cronjob-3.png?raw=true)

Grafana dashbord 的JSON格式配置文件 [K8s定时任务监控-Grafana.json](https://github.com/chenguolin/chenguolin.github.io/blob/master/data/grafana/K8s%E5%AE%9A%E6%97%B6%E4%BB%BB%E5%8A%A1%E7%9B%91%E6%8E%A7-Grafana.json)，使用Grafana Import即可恢复dashbord。

针对以上 Grafana 报表，具体的配置如下。注意，Prometheus 会自动添加 job 和 instance 两个label，其它label可以在 Prometheus server 采集配置文件里配置，或业务自行定义上报。

## ① Variables
1. datasource (数据源)
    ```
    1). General
        * Type: Datasource
        * Label: Datasource
    2). Data source options
        * Type: Prometheus
        * Instance: thanos  //https://github.com/thanos-io/thanos  要求通过Thanos部署Prometheus集群  http://dockone.io/article/6019
    ```
2. cluster (集群)
    ```
    1). General
        * Type: Query
        * Label: cluster
    2). Query Options
        * Data source: $datasource
        * Query: label_values(k8s_batch_total_job_completed{job="k8s-batch-metrics"}, cluster)    //cluster 字段可以在 Prometheus Server采集配置文件里面配置
    ```
3. namespace (命名空间)
    ```
    1). General
        * Type: Query
        * Label: namespace
    2). Query Options
        * Data source: $datasource
        * Query: label_values(k8s_batch_total_job_completed{job="k8s-batch-metrics", cluster=~"$cluster"}, namespace)
    ```
4. cronjob (定时任务)
    ```
    1). General
        * Type: Query
        * Label: cronjob
    2). Query Options
        * Data source: $datasource
        * Query: label_values(k8s_batch_total_job_completed{job="k8s-batch-metrics", cluster=~"$cluster", namespace=~"$namespace"}, cronjob)
    ```
5. Interval (一般情况下都需要设置Interval字段用来筛选数据)
    ```
    1). General
        * Type: Interval
        * Label: Interval
    2). Interval options
        * Values: 1m,2m,10m,2h
        * Auto Option: 开
        * Min Interval: 2m   //一般设置为Prometheus server抓取频率的2倍，如果是1m抓取一次metrics，则设置为2m
    ```
    
## ② 总览
1. Running CronJob (Singlestat)
    ```
    sql: sum(k8s_batch_total_cronjob_running{job=~"k8s-batch-metrics",cluster=~"$cluster",namespace=~"$namespace",cronjob=~"$cronjob"})
    Instant: 开  //获取的是当前最新的时序数据
    ```
2. Suspend CronJob (Singlestat)
    ```
    sql: sum(k8s_batch_total_cronjob_suspend{job=~"k8s-batch-metrics",cluster=~"$cluster",namespace=~"$namespace",cronjob=~"$cronjob"})
    Instant: 开  //获取的是当前最新的时序数据
    ```
3. Running Job (Singlestat)
    ```
    sql: sum(k8s_batch_total_job_running{job=~"k8s-batch-metrics",cluster=~"$cluster",namespace=~"$namespace",cronjob=~"$cronjob"})
    Instant: 开  //获取的是当前最新的时序数据
    ```
4. Completed Job (Singlestat)
    ```
    sql: sum(k8s_batch_total_job_completed{job=~"k8s-batch-metrics",cluster=~"$cluster",namespace=~"$namespace",cronjob=~"$cronjob"})
    Instant: 开  //获取的是当前最新的时序数据
    ```
5. Failed Job (Singlestat)
    ```
    sum(k8s_batch_total_job_failed{job=~"k8s-batch-metrics",cluster=~"$cluster",namespace=~"$namespace",cronjob=~"$cronjob"})
    Instant: 开  //获取的是当前最新的时序数据
    ```
6. CronJob 分布 (Pie Chart)
    ```
    Running sql: sum(k8s_batch_total_cronjob_running{job=~"k8s-batch-metrics",cluster=~"$cluster",namespace=~"$namespace",cronjob=~"$cronjob"})
    Suspend sql: sum(k8s_batch_total_cronjob_suspend{job=~"k8s-batch-metrics",cluster=~"$cluster",namespace=~"$namespace",cronjob=~"$cronjob"})
    Instant: 开  //获取的是当前最新的时序数据
    ```
7. Job 分布 (Pie Chart)
    ```
    Running sql: sum(k8s_batch_total_job_running{job=~"k8s-batch-metrics",cluster=~"$cluster",namespace=~"$namespace",cronjob=~"$cronjob"})
    Completed sql: sum(k8s_batch_total_job_completed{job=~"k8s-batch-metrics",cluster=~"$cluster",namespace=~"$namespace",cronjob=~"$cronjob"})
    Failed sql: sum(k8s_batch_total_job_failed{job=~"k8s-batch-metrics",cluster=~"$cluster",namespace=~"$namespace",cronjob=~"$cronjob"})
    Instant: 开  //获取的是当前最新的时序数据
    ```
 8. CronJob (Graph)
    ```
    Running sql: sum(k8s_batch_total_cronjob_running{job=~"k8s-batch-metrics",cluster=~"$cluster",namespace=~"$namespace",cronjob=~"$cronjob"})
    Suspend sql: sum(k8s_batch_total_cronjob_suspend{job=~"k8s-batch-metrics",cluster=~"$cluster",namespace=~"$namespace",cronjob=~"$cronjob"})
    ```
9. Job (Graph)
    ```
    Running sql: sum(k8s_batch_total_job_running{job=~"k8s-batch-metrics",cluster=~"$cluster",namespace=~"$namespace",cronjob=~"$cronjob"})
    Completed sql: sum(k8s_batch_total_job_completed{job=~"k8s-batch-metrics",cluster=~"$cluster",namespace=~"$namespace",cronjob=~"$cronjob"})
    Failed sql: sum(k8s_batch_total_job_failed{job=~"k8s-batch-metrics",cluster=~"$cluster",namespace=~"$namespace",cronjob=~"$cronjob"})
    ```

## ③ 资源消耗
1. Running Job Request CPU (Singlestat)
    ```
    sql: sum(k8s_batch_running_job_req_cpu{job=~"k8s-batch-metrics",cluster=~"$cluster",namespace=~"$namespace",cronjob=~"$cronjob"})
    Instant: 开  //获取的是当前最新的时序数据
    ```
2. Running Job Limit CPU (Singlestat)
    ```
    sql: sum(k8s_batch_running_job_limit_cpu{job=~"k8s-batch-metrics",cluster=~"$cluster",namespace=~"$namespace",cronjob=~"$cronjob"})
    Instant: 开  //获取的是当前最新的时序数据
    ```
3. Running Job Request Memory (Singlestat)
    ```
    sql: sum(k8s_batch_running_job_req_mem{job=~"k8s-batch-metrics",cluster=~"$cluster",namespace=~"$namespace",cronjob=~"$cronjob"})
    Instant: 开  //获取的是当前最新的时序数据
    ```

4. Running Job Limit Memory (Singlestat)
    ```
    sql: sum(k8s_batch_running_job_limit_mem{job=~"k8s-batch-metrics",cluster=~"$cluster",namespace=~"$namespace",cronjob=~"$cronjob"})
    Instant: 开  //获取的是当前最新的时序数据
    ```

5. Running Job CPU (Graph)
    ```
    Req sql: sum(k8s_batch_running_job_req_cpu{job=~"k8s-batch-metrics",cluster=~"$cluster",namespace=~"$namespace",cronjob=~"$cronjob"})
    Limit sql: sum(k8s_batch_running_job_limit_cpu{job=~"k8s-batch-metrics",cluster=~"$cluster",namespace=~"$namespace",cronjob=~"$cronjob"})
    ```

6. Running Job Memory (Graph)
    ```
    Req sql: sum(k8s_batch_running_job_req_mem{job=~"k8s-batch-metrics",cluster=~"$cluster",namespace=~"$namespace",cronjob=~"$cronjob"})
    Limit sql: sum(k8s_batch_running_job_limit_mem{job=~"k8s-batch-metrics",cluster=~"$cluster",namespace=~"$namespace",cronjob=~"$cronjob"})
    ```

## ④ 定时任务
1. Running CronJob 创建时间分布 (Pie Chart)
    ```
    < 1d sql: count(k8s_batch_running_cronjob_starttime_bucket{job=~"k8s-batch-metrics",cluster=~"$cluster",namespace=~"$namespace",cronjob=~"$cronjob",le="86400"} > 0 )
    1d ~ 3d sql: count(k8s_batch_running_cronjob_starttime_bucket{job=~"k8s-batch-metrics",cluster=~"$cluster",namespace=~"$namespace",cronjob=~"$cronjob",le="259200"} - ignoring(le) k8s_batch_running_cronjob_starttime_bucket{job=~"k8s-batch-metrics",cluster=~"$cluster",namespace=~"$namespace",cronjob=~"$cronjob",le="86400"} != 0)
    3d ~ 7d sql: count(k8s_batch_running_cronjob_starttime_bucket{job=~"k8s-batch-metrics",cluster=~"$cluster",namespace=~"$namespace",cronjob=~"$cronjob",le="604800"} - ignoring(le) k8s_batch_running_cronjob_starttime_bucket{job=~"k8s-batch-metrics",cluster=~"$cluster",namespace=~"$namespace",cronjob=~"$cronjob",le="259200"} != 0)
    7d ~ 15d sql: count(k8s_batch_running_cronjob_starttime_bucket{job=~"k8s-batch-metrics",cluster=~"$cluster",namespace=~"$namespace",cronjob=~"$cronjob",le="1.296e+06"} - ignoring(le) k8s_batch_running_cronjob_starttime_bucket{job=~"k8s-batch-metrics",cluster=~"$cluster",namespace=~"$namespace",cronjob=~"$cronjob",le="604800"} != 0)
    15d ~ 30d sql: count(k8s_batch_running_cronjob_starttime_bucket{job=~"k8s-batch-metrics",cluster=~"$cluster",namespace=~"$namespace",cronjob=~"$cronjob",le="2.592e+06"} - ignoring(le) k8s_batch_running_cronjob_starttime_bucket{job=~"k8s-batch-metrics",cluster=~"$cluster",namespace=~"$namespace",cronjob=~"$cronjob",le="1.296e+06"} != 0)
    > 30d sql: count(k8s_batch_running_cronjob_starttime_bucket{job=~"k8s-batch-metrics",cluster=~"$cluster",namespace=~"$namespace",cronjob=~"$cronjob",le="+Inf"} - ignoring(le) k8s_batch_running_cronjob_starttime_bucket{job=~"k8s-batch-metrics",cluster=~"$cluster",namespace=~"$namespace",cronjob=~"$cronjob",le="2.592e+06"} != 0)
    Instant: 开  //获取的是当前最新的时序数据
    ```
2. Running Job 创建时间分布 (Pie Chart)
    ```
    < 1m sql: count(k8s_batch_running_job_starttime_bucket{job=~"k8s-batch-metrics",cluster=~"$cluster",namespace=~"$namespace",cronjob=~"$cronjob",le="60"})
    1 ~ 10m sql: count(k8s_batch_running_job_starttime_bucket{job=~"k8s-batch-metrics",cluster=~"$cluster",namespace=~"$namespace",cronjob=~"$cronjob",le="600"} - k8s_batch_running_job_starttime_bucket{job=~"k8s-batch-metrics",cluster=~"$cluster",namespace=~"$namespace",cronjob=~"$cronjob",le="60"} != 0)
    10 ~ 60m sql: count(k8s_batch_running_job_starttime_bucket{job=~"k8s-batch-metrics",cluster=~"$cluster",namespace=~"$namespace",cronjob=~"$cronjob",le="3600"} - k8s_batch_running_job_starttime_bucket{job=~"k8s-batch-metrics",cluster=~"$cluster",namespace=~"$namespace",cronjob=~"$cronjob",le="60"} != 0)
    1 ~ 6h sql: count(k8s_batch_running_job_starttime_bucket{job=~"k8s-batch-metrics",cluster=~"$cluster",namespace=~"$namespace",cronjob=~"$cronjob",le="21600"} - k8s_batch_running_job_starttime_bucket{job=~"k8s-batch-metrics",cluster=~"$cluster",namespace=~"$namespace",cronjob=~"$cronjob",le="3600"} != 0)
    6 ~ 12h sql: count(k8s_batch_running_job_starttime_bucket{job=~"k8s-batch-metrics",cluster=~"$cluster",namespace=~"$namespace",cronjob=~"$cronjob",le="43200"} - k8s_batch_running_job_starttime_bucket{job=~"k8s-batch-metrics",cluster=~"$cluster",namespace=~"$namespace",cronjob=~"$cronjob",le="21600"} != 0)
    12~24h sql: count(k8s_batch_running_job_starttime_bucket{job=~"k8s-batch-metrics",cluster=~"$cluster",namespace=~"$namespace",cronjob=~"$cronjob",le="86400"} - k8s_batch_running_job_starttime_bucket{job=~"k8s-batch-metrics",cluster=~"$cluster",namespace=~"$namespace",cronjob=~"$cronjob",le="43200"} != 0)
    >24h sql: count(k8s_batch_running_job_starttime_bucket{job=~"k8s-batch-metrics",cluster=~"$cluster",namespace=~"$namespace",cronjob=~"$cronjob",le="+Inf"} - k8s_batch_running_job_starttime_bucket{job=~"k8s-batch-metrics",cluster=~"$cluster",namespace=~"$namespace",cronjob=~"$cronjob",le="86400"} != 0)
    Instant: 开  //获取的是当前最新的时序数据
    ```
3. Running CronJob 创建时间分布 (Table)
   同1

# 四. Prometheus告警
Prometheus 自带告警模块，允许用户自定义告警规则，可以发送到邮件、企业微信，还可以自定义接口发送到短信，非常方便。具体可以参考  https://prometheus.io/docs/alerting/overview/




