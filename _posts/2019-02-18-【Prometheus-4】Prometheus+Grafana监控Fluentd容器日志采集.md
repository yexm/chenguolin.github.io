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
    
# 三. Grafana配置
## ① 


