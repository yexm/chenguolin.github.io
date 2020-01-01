---
layout:  post  # 使用的布局（不需要改）
catalog: true  # 是否归档
author: 陈国林 # 作者
tags:          #标签
    - Kubernetes
---

# 一. 概述
我们可以使用 `docker logs` 命令查看当前正在运行的容器的日志，默认情况下会通过终端打印出日志。Linux和类Unix系统默认会打开3个流，`stdint`、`stdout`和 `stderr`，默认情况下 `docker logs` 命令会打印 `stdout`和`stderr`的日志。

在以下这些场景 `docker logs` 命令不会打印容器日志

1. 我们通过配置 `logging driver` 把日志发送到文件 或 外部存储
2. 容器进程并不是把日志打印到 `stdout`和`stderr`，而是输出到容器内文件

Docker包含多种日志实现机制，这些机制我们称为 `logging drivers`，Docker目前支持以下几种类型 logging drivers [docker logging](https://docs.docker.com/config/containers/logging/configure/)，默认情况下Docker会使用 json-file 做为logging drivers。相关的源代码可以参考 [docker daemon logger](https://github.com/moby/moby/tree/master/daemon/logger)

| Driver     | 描述 |
| ---------- | --- |
| none       | 关闭日志功能，docker logs 命令看不到任何输出，参考源码 [daemon logs.go](https://github.com/moby/moby/blob/master/daemon/logs.go#L44) |
| [local](https://docs.docker.com/config/containers/logging/local/)      |  接管了容器stdout，把日志写到宿主机 /var/lib/docker/containers/{container-id}/local-logs 目录 |
| [json-file](https://docs.docker.com/config/containers/logging/json-file/)  |  接管了容器stdout，以JSON格式把日志写到宿主机 /var/lib/docker/containers/{container-id}/{container-id}-json.log |
| [syslog](https://docs.docker.com/config/containers/logging/syslog/)     | 接管了容器stdout，把日志写到宿主机syslog，要求宿主机运行syslog  |
| [journald](https://docs.docker.com/config/containers/logging/journald/)   |  接管了容器stdout，把日志写到宿主机journald，要求宿主机运行journald |
| [gelf](https://docs.docker.com/config/containers/logging/gelf/)       |  接管了容器stdout，把日志写到Graylog 或 Logstash |
| [fluentd](https://docs.docker.com/config/containers/logging/fluentd/)    |  接管了容器stdout，把日志做为fluentd采集输入源，要求宿主机运行fluentd|
| [awslogs](https://docs.docker.com/config/containers/logging/awslogs/)    |  接管了容器stdout，把日志写到Amazon CloudWatch Logs |
| [splunk](https://docs.docker.com/config/containers/logging/splunk/)     |  接管了容器stdout，把日志写到splunk |
| [etwlogs](https://docs.docker.com/config/containers/logging/etwlogs/)    |  接管了容器stdout，把日志写到etwlogs，只允许在Windows平台使用 |
| [gcplogs](https://docs.docker.com/config/containers/logging/gcplogs/)    |  接管了容器stdout，把日志写到Google Cloud Platform (GCP) Logging |
| [logentries](https://docs.docker.com/config/containers/logging/logentries/) |  接管了容器stdout，把日志写到Rapid7 Logentries |

Docker daemon启动的时候可以设置 logging driver，我们可以通过 `ps -axu` 命令查看dockerd进程，确认daemon.json配置文件的路径，通过在daemon.json配置logging driver，例如我们使用 json-file 做为logging driver。这样默认情况下所有的容器都会使用 json-file 做为logging driver。

```
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3",
    "labels": "production_status",
    "env": "os,customer"
  }
}
```

除了可以给Docker daemon配置logging driver之外，我们也可以给某个容器配置。当要运行容器的时候，我们可以通过 `--log-driver` 选项进行配置不同的logging driver

```
docker run --log-driver json-file --log-opt max-size=10m alpine echo hello world
```

# 二. Docker日志实现
对于业务容器，我们分成2种情况。一种是业务容器内日志文件，这种情况Docker daemon 不可能也不应该深入应用内部，截获并接管日志文件内容，这只会破坏 Docker 的通用性。另外一种是业务输出到stdout日志，这种情况上文我们说到Docker daemon会通过 logging driver 接管stdout，并根据具体的logging driver进行日志分发。

![](https://github.com/chenguolin/chenguolin.github.io/blob/master/data/image/docker-logging-driver.png?raw=true)

`实现原理: Docker Daemon 在运行容器时会创建一个 goroutine，负责接管容器 stdout 日志。由于此 goroutine 绑定了整个容器内所有进程的 stdout 文件描述符，因此容器内应用的所有 stdout 日志都会被 goroutine 接收。goroutine 接收到容器的 stdout 日志时，立即根据对应的 logging driver 进行日志分发。默认使用 json-file 则会把日志按 json 格式写入到 /var/lib/docker/containers/{container-id}/{container_id}-json.log 文件，并根据配置进行滚动切割和压缩。`

关于 json-file 的
