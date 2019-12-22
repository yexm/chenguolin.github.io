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
