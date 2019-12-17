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

# 三. Grafana配置
![](https://github.com/chenguolin/chenguolin.github.io/blob/master/data/image/http-api-prometheus-grafana-1.png?raw=true)

![](https://github.com/chenguolin/chenguolin.github.io/blob/master/data/image/http-api-prometheus-grafana-2.png?raw=true)

![](https://github.com/chenguolin/chenguolin.github.io/blob/master/data/image/http-api-prometheus-grafana-3.png?raw=true)


# 四. Prometheus告警
Prometheus 自带告警模块，允许用户自定义告警规则，可以发送到邮件、企业微信，还可以自定义接口发送到短信，非常方便。具体可以参考 https://prometheus.io/docs/alerting/overview/

