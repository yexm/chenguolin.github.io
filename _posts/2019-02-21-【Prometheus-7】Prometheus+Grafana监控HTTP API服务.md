---
layout:  post  # 使用的布局（不需要改）
catalog: true  # 是否归档
author: 陈国林 # 作者
tags:          #标签
    - Prometheus
---

# 一. 背景
`Prometheus`做为CNCF第二名成员，事实上已经是云原生生态监控的标准。云原生生态体系绝大多数的开源组件都是使用Golang进行编写，例如Kubernetes项目。因此，使用`Prometheus`监控Golang程序也就成了开发必须要掌握的一个技能。`Prometheus`虽然是监控的标准，但在报表的展示上却需要`Grafana`配合，实际上`Prometheus + Grafana`的监控方案已经是业内公认的一个标准。

这篇文章会总结下如何使用 `Prometheus + Grafana` 监控 Golang HTTP API 服务，包括实现细节、报表配置 以及报警配置。

# 二. 监控实现
HTTP API 服务是非常常见的一种提供在线服务方式，对于 Golang 来说也有好几种 HTTP 框架，我习惯用的是 `Gin`。下面介绍一下使用 Gin 开发 HTTP API 服务如何使用 Prometheus 进行相关的监控。具体可以参考 [go-api-service prometheus](https://github.com/chenguolin/go-api-service)

Gin 可以支持用户设置一序列的 middleware，我们可以在 middleware 处进行 http 请求的指标统计 [middleware](https://github.com/chenguolin/go-api-service/blob/master/cmd/middleware.go)

```
// AccessLogMiddleware middleware func to print http request access log
func AccessLogMiddleware() gin.HandlerFunc {
	return func(c *gin.Context) {
		// all devops request not need print log
		if strings.Contains(c.Request.URL.Path, "devops") {
			c.Next()
			return
		}

		start := time.Now()
		reqBody, _ := ioutil.ReadAll(c.Request.Body)
		c.Request.Body = ioutil.NopCloser(bytes.NewBuffer(reqBody))
		path := c.Request.URL.Path
		query := c.Request.URL.RawQuery
		clientIP := c.ClientIP()
		method := c.Request.Method
		unescapeReqbody, _ := url.PathUnescape(string(reqBody))

		// HTTP request start
		log.Info("HTTP request start ...",
			log.String(request.RequestIDKey, request.GetRequestID(c)),
			log.String("path", path),
			log.String("query", query),
			log.String("ip", clientIP),
			log.String("method", method),
			log.String("body", unescapeReqbody))

		// process next handle
		c.Next()

		// HTTP request end
		end := time.Now()
		latency := end.Sub(start).Seconds()
		statusCode := c.Writer.Status()

		// check status code
		log.Info("HTTP request end ~",
			log.String(request.RequestIDKey, request.GetRequestID(c)),
			log.Float64("latency", latency),
			log.Int("status_code", statusCode))

		// add prometheus metrics
		// TODO cluster and instance need 2 set true value
		prometheus.SetHTTPRequestLatencyMetrics("cluster", "instance",
			path, method, statusCode, latency)
	}
}
```

Prometheus 相关的 metrics 值设置可以通过封装独立的 pkg 来实现，例如 [pkg/prometheus/metrics.go](https://github.com/chenguolin/go-api-service/tree/master/pkg/prometheus/metrics.go)
```
package prometheus

import (
	"strconv"

	"github.com/chenguolin/go-prometheus/prometheus"
)

var (
	promCounterVecs   = make(map[string]*prometheus.CounterVec)
	promHistogramVecs = make(map[string]*prometheus.HistogramVec)
	// TODO other type
)

func init() {
	promCounterVecs["http_request_error_count_metrics"] = prometheus.NewCounterVec(
		"http_request_error_count_metrics",
		"http request error count monitor",
		[]string{"cluster", "instance", "path", "method", "status_code"})
	promCounterVecs["http_request_success_count_metrics"] = prometheus.NewCounterVec(
		"http_request_success_count_metrics",
		"http request success count monitor",
		[]string{"cluster", "instance", "path", "method", "status_code"})
	promHistogramVecs["http_request_latency_metrics"] = prometheus.NewHistogramVec(
		"http_request_latency_metrics",
		"http request latency metrics",
		[]float64{10, 50, 100, 500, 1000},
		[]string{"cluster", "instance", "path", "method", "status_code"})
}

// SetHTTPRequestSuccessCountMetrics set http request success count metrics
func SetHTTPRequestSuccessCountMetrics(cluster, instance, path, method string, statusCode int) {
	promLabels := make(prometheus.Labels)
	promLabels["cluster"] = cluster
	promLabels["instance"] = instance
	promLabels["path"] = path
	promLabels["method"] = method
	promLabels["status_code"] = strconv.Itoa(statusCode)

	// set metrics
	promCounterVecs["http_request_success_count_metrics"].Add(promLabels, 1)
}

// SetHTTPRequestErrorCountMetrics set http request error count metrics
func SetHTTPRequestErrorCountMetrics(cluster, instance, path, method string, statusCode int) {
	promLabels := make(prometheus.Labels)
	promLabels["cluster"] = cluster
	promLabels["instance"] = instance
	promLabels["path"] = path
	promLabels["method"] = method
	promLabels["status_code"] = strconv.Itoa(statusCode)

	// set metrics
	promCounterVecs["http_request_error_count_metrics"].Add(promLabels, 1)
}

// SetHTTPRequestLatencyMetrics set http request latency metrics
func SetHTTPRequestLatencyMetrics(cluster, instance, path, method string, statusCode int, latency float64) {
	promLabels := make(prometheus.Labels)
	promLabels["cluster"] = cluster
	promLabels["instance"] = instance
	promLabels["path"] = path
	promLabels["method"] = method
	promLabels["status_code"] = strconv.Itoa(statusCode)

	// set metrics
	promHistogramVecs["http_request_latency_metrics"].Observe(promLabels, latency)
}
```

# 三. Grafana配置
![](https://github.com/chenguolin/chenguolin.github.io/blob/master/data/image/http-api-prometheus-grafana-1.png?raw=true)
![](https://github.com/chenguolin/chenguolin.github.io/blob/master/data/image/http-api-prometheus-grafana-2.png?raw=true)
![](https://github.com/chenguolin/chenguolin.github.io/blob/master/data/image/http-api-prometheus-grafana-3.png?raw=true)

# 四. Prometheus告警
Prometheus 自带告警模块，允许用户自定义告警规则，可以发送到邮件、企业微信，还可以自定义接口发送到短信，非常方便。具体可以参考 https://prometheus.io/docs/alerting/overview/

