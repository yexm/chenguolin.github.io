---
layout:  post  # 使用的布局（不需要改）
catalog: true  # 是否归档
author: 陈国林 # 作者
tags:          #标签
    - Prometheus
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







