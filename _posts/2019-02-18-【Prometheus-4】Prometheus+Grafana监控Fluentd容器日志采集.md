---
layout:  post  # 使用的布局（不需要改）
catalog: true  # 是否归档
author: 陈国林 # 作者
tags:          #标签
    - Prometheus
---

# 一. 背景
Prometheus 做为CNCF第二名成员，事实上已经是云原生生态监控的标准。云原生生态体系绝大多数的开源组件都是使用Golang进行编写，例如Kubernetes项目。Prometheus虽然是监控的标准，但在报表的展示上却需要Grafana配合，因此 Prometheus + Grafana的监控方案已经是业内公认的一个标准。

这篇文章会总结下如何使用 Prometheus + Grafana 监控 Fluentd 容器日志采集，包括实现细节、报表配置 以及报警配置。

# 二. 监控实现
k8s fluentd 项目可以参考 [k8s-fluentd](https://github.com/chenguolin/k8s-fluentd)，fluentd 监控实现分为2个部分，一部分是fluentd自带的插件会使用 Prometheus进行相关的数据统计，另外一部分是需要业务自行编写插件实现。

Prometheus 通过服务发现功能获取资源暴露的监控点，主动拉取所有监控数据，具体包括集群所有pod，service-endpoints，node的kubelet及其cadvisor的metric信息。业务只需要把相关的 `/metrics` 接口暴露出来即可，Prometheus server会自动发现并定期拉取数据。

1. fluentd自带插件可以参考 [Fluentd-Prometheus监控](https://chenguolin.github.io/2019/03/05/Fluentd-6-Fluentd-Prometheus%E7%9B%91%E6%8E%A7/)，主要是以下几个插件
    + [in_prometheus](https://github.com/fluent/fluent-plugin-prometheus/blob/master/lib/fluent/plugin/in_prometheus.rb): 插件用于暴露监控metrics指标，提供HTTP接口供Prometheus服务采集
    + [in_prometheus_monitor](https://github.com/fluent/fluent-plugin-prometheus/blob/master/lib/fluent/plugin/in_prometheus_monitor.rb): 插件用于Fluentd Output带buffer插件监控
    + [in_prometheus_output_monitor](https://github.com/fluent/fluent-plugin-prometheus/blob/master/lib/fluent/plugin/in_prometheus_output_monitor.rb): 插件用于Fluentd Ouput插件监控，比in_prometheus_monitor插件提供更多metrics指标
    + [in_prometheus_tail_monitor](https://github.com/fluent/fluent-plugin-prometheus/blob/master/lib/fluent/plugin/in_prometheus_tail_monitor.rb): 插件用于Fluentd in_tail插件监控
    + [filter_prometheus](https://github.com/fluent/fluent-plugin-prometheus/blob/master/lib/fluent/plugin/filter_prometheus.rb): 插件用于records相关统计，例如可以用于统计总输入records数
    + [out_prometheus](https://github.com/fluent/fluent-plugin-prometheus/blob/master/lib/fluent/plugin/out_prometheus.rb): 插件用于records相关统计，例如可以用于统计总输出records数
2. 业务自行编写的监控插件 可以参考 [k8s-fluentd plugins](https://github.com/chenguolin/k8s-fluentd/tree/master/plugins)，主要有以下几个插件
    + [out_flowcounter](https://github.com/chenguolin/k8s-fluentd/blob/master/plugins/out_flowcounter.rb): 插件用于统计output records、output bytes等指标
    + [out_fluentd_monitor](https://github.com/chenguolin/k8s-fluentd/blob/master/plugins/out_fluentd_monitor.rb): 插件用于统计fluentd进程自身的指标，例如error log、write kafka failed 等指标
    
k8s集群pod和service-endpoints常用字段如下，需要业务在annotations里面加上这些字段
```
prometheus_io_scrape: annotation中增加这个字段为true后才能被Prometheus监控
prometheus_io_scheme: Prometheus拉取协议，https或者http
prometheus_io_path: Prometheus拉取监控数据的路径，默认为/metrics
prometheus_io_port: Prometheus拉取监控数据的端口号
```
    
# 三. Grafana配置
## ① Fluentd采集监控
Grafana dashbord 的JSON格式配置文件 [Fluentd-采集监控-Grafana.json](https://github.com/chenguolin/chenguolin.github.io/blob/master/data/grafana/Fluentd-%E9%87%87%E9%9B%86%E7%9B%91%E6%8E%A7-Grafana.json)，使用Grafana Import即可恢复dashbord。

![](https://github.com/chenguolin/chenguolin.github.io/blob/master/data/image/fluentd-prometheus-grafana-1.png?raw=true)
![](https://github.com/chenguolin/chenguolin.github.io/blob/master/data/image/fluentd-prometheus-grafana-2.png?raw=true)
![](https://github.com/chenguolin/chenguolin.github.io/blob/master/data/image/fluentd-prometheus-grafana-3.png?raw=true)
![](https://github.com/chenguolin/chenguolin.github.io/blob/master/data/image/fluentd-prometheus-grafana-4.png?raw=true)

## ② Fluentd进程监控
Grafana dashbord 的JSON格式配置文件 [Fluentd-进程监控-Grafana.json](https://github.com/chenguolin/chenguolin.github.io/blob/master/data/grafana/Fluentd-%E8%BF%9B%E7%A8%8B%E7%9B%91%E6%8E%A7-Grafana.json)，使用Grafana Import即可恢复dashbord。

![](https://github.com/chenguolin/chenguolin.github.io/blob/master/data/image/fluentd-process-prometheus-grafana-1.png?raw=true)
![](https://github.com/chenguolin/chenguolin.github.io/blob/master/data/image/fluentd-process-prometheus-grafana-2.png?raw=true)
![](https://github.com/chenguolin/chenguolin.github.io/blob/master/data/image/fluentd-process-prometheus-grafana-3.png?raw=true)

# 四. 监控报警
Prometheus 自带告警模块，允许用户自定义告警规则，可以发送到邮件、企业微信，还可以自定义接口发送到短信，非常方便。具体可以参考 https://prometheus.io/docs/alerting/overview/




