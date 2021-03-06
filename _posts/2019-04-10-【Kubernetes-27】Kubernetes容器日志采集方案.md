---
layout:  post  # 使用的布局（不需要改）
catalog: true  # 是否归档
author: 陈国林 # 作者
tags:          #标签
    - Kubernetes
---

# 一. 背景
每个应用程序都会打印日志，通过日志可以帮助我们了解应用程序内部发生了什么，日志也被用于调试问题和监控应用程序，日志使得应用程序运行的动作变得透明。

传统的应用程序日志有以下几个特点

1. 传统应用程序是直接部署在一个或多个物理机(虚拟机)上，业务按单体或微服务方式进行部署。
2. 应用程序按照类型把日志输出到物理机指定目录文件，并进行滚动切割。
3. 业务日志格式随意，没有统一方式
4. 业务通过登录部署服务器，进行日志查看和问题排查
5. 业务运维使用工具进行日志压缩、备份和清理

由上面几个点我们知道，传统日志架构基本是为开发或运维提供排查问题的信息，是面向人去设计的，人脑天生适合处理非结构化的数据，因此日志信息非常随意。

但是随着微服务、容器化技术的发展促使我们的应用部署架构发生了变化，业务容器化后会面临以下几个问题
1. 业务Pod部署所在的宿主机不定，登录服务器查看日志方式不太可行
2. 业务Pod有生命周期，当Pod销毁的时候日志也就会丢失 
3. 传统的使用日志进行统计、报警、分析等行为均会受限
4. 由于容器的隔离性，日志文件通常是对外不可访问的
5. 容器存储空间有限，长时间的写入日志文件，也可能造成容器存储空间打满，导致容器挂掉
6. 业务容器化后日志架构面临诸多挑战 

由上可知，传统的日志架构已经不再适用，在业务全面容器化的时代，我们亟需找到一种新的日志架构来适应容器化环境。

我们期望日志的输出具有统一的结构化的格式，服务将日志输出到标准输出，不需要关心日志如何落地。运行环境截获日志并由统一的日志收集Agent将日志过滤，处理，采集，通过日志传输组件（比如kafka）分发给不同处理程序。

# 二. Kubernetes日志架构
大部分现代应用都有各种日志机制，因此，大部分容器引擎也被设计支持各种日志。对于容器化的应用来说，最简单也最推荐的日志采集方法是将日志写到标准输出和标准错误输出。

在k8s中，Pod所在的机器并不固定，无法直接查看日志，同时日志会随着Pod的重启而丢失。为了适应这种情况，需要将日志收集到统一的日志中心，以便查询和分析。根据日志的输出形式可以将日志划分为 `标准输出日志`、`文件输出日志`

## ① 标准输出日志
容器化应用通过stdout和stderr输出日志，然后由容器引擎处理，例如Docker默认使用JSON file做为log-driver，通过接管stdout和stderr把日志写到JSON文件中。

Docker在启动的过程中，会接管容器的标准输出，然后交给log-driver处理，这些日志会写到文件`/var/lib/docker/container/{container_id}/xxxx.log`路径下，K8s会把`/var/lib/docker/container/{container_id}/xxxx.log`这个文件软链到 `/var/log/containers` 目录下，可以在每个K8s worker节点上启动采集Agent，采集`/var/log/containers`目录下日志文件。在k8s中，一般通过DeamonSet保证每个worker节点都启动一个采集Agent。

![](https://github.com/chenguolin/chenguolin.github.io/blob/master/data/image/k8s-logging-with-node-agent.png?raw=true)

## ② 文件输出日志
对于那些没有办法输出到stdout和stderr的日志，还是继续输出到文件，只不过日志文件由于是在容器内，由于隔离性存在容器内文件并不能被直接访问，因此我们会使用
sidecar容器，读取文件内容并输出到标准输出，这样就可以通过标准输出的流程收集这些日志。

![](https://github.com/chenguolin/chenguolin.github.io/blob/master/data/image/k8s-logging-with-streaming-sidecar.png?raw=true)

# 三. 采集方案
## ① 业务需求
业务对日志的需求归纳总结起来就是以下几点

1. 日志查看 (grep命令，需要有全文检索功能)
2. 日志报警 (根据ERROR日志告警)
3. 日志统计 (根据日志统计各种信息)
4. 命令行查看日志
5. 离线计算和分析

## ② 采集核心诉求
针对业务的需求，日志采集这边的核心诉求归纳总结起来主要是以下几点

1. 日志不丢
2. 日志不会重复采集
3. 时效性强
4. 采集流程可定制
5. 具有日志所属K8s相关meta信息，唯一标识
6. 支持监控告警

## ③ 标准输出日志采集
根据Kubernetes日志架构方案，以及业务需求和采集的核心诉求，`标准输出日志采集`方案为 `Fluentd + Kafka + ELK`，通过Fluentd进行采集，数据写入Kafka不同topic，通过Logstash消费写入ES，最终通过Kibana进行展示查看。

1. 使用Fluentd做为每个K8s worker节点的采集Agent
    + 社区活跃: CNCF官方项目 ，Kubernetes官方推荐采集器
    + 扩展性强: 源码开源，配置简单，通过插件化形式来管理数据处理流程，650+开源插件
    + 性能好: C和Ruby编写，极简模式下单进程采集能力15MB/s，支持多进程，性能可线性扩展
2. 使用Kafka做为消息队列，时效性强，当ES写入有瓶颈时，可以用来缓存数据
3. 使用ELK做为数据消费、存储、展示
    + Elasticsearch: 基于Lucene的全文检索引擎，存储容量大，保存周期长
    + 使用Kibana支持高级搜索功能，日志查看很方便
    + 还能基于Kibana做离线统计、分析

![](https://github.com/chenguolin/chenguolin.github.io/blob/master/data/image/k8s-logging-stdout.png?raw=true)

## ④ 文件输出日志采集
根据Kubernetes日志架构方案，以及业务需求和采集的核心诉求，`文件输出日志采集`方案为 `sidecar容器 + Fluentd + Kafka + ELK`，通过sidecar容器导出到标准输出后，使用Fluentd进行采集，数据写入Kafka不同topic，通过Logstash消费写入ES，最终通过Kibana进行展示查看。

# 四. 组件介绍
## ① Fluentd组件
Fluentd是一个开源的日志采集器，是CNCF毕业项目，社区活跃。它的原理是日志使用input插件读取后，使用一序列的插件进行处理，最后通过output插件输出。我们使用Fluentd v0.12.x版本，使用K8s configmap生成Fluentd的配置文件。通过K8s Daemonset部署在每台worker节点上，Pod内容器运行6个Fluentd进程。Fluentd配置文件配置采集`/var/log/containers`目录以及k8s组件的一些日志，日志采集完输出到Kafka。

Fluentd提出了 `统一日志层（unified-logging-layer）`的概念，其基本思想是将日志抽象为Producer和Consumer，Producer，Consumer 可以有多种多样，Fluentd通过插件系统来适配不同的Producer和Consumer。

![](https://github.com/chenguolin/chenguolin.github.io/blob/master/data/image/fluentd-new-architecture.png?raw=true)

目前日志的处理流程如下所示，使用`in_tail`插件读取后，经过一序列的插件处理，最后使用`kafka_buffered`插件进行输出。

![](https://github.com/chenguolin/chenguolin.github.io/blob/master/data/image/k8s-logging-fluentd-pipeline.png?raw=true)

1. detect_exceptions: [fluent-plugin-detect-exceptions](https://github.com/GoogleCloudPlatform/fluent-plugin-detect-exceptions)，识别日志是否属于异常信息，并合并这些异常日志
2. concatdocker: [fluent-plugin-concat](https://github.com/fluent-plugins-nursery/fluent-plugin-concat)，由于JSON file限制单行最大为16K，因此当业务单行输出的日志超过16K之后会被JSON file切割为多行，需要进行拼接
3. prune_log: 日志裁剪，由于写入Kafka单条消息不能超过`1M`，因此超过单行超过1M就进行裁剪
4. kubernetes_metadata: [fluent-plugin-kubernetes_metadata](https://github.com/fabric8io/fluent-plugin-kubernetes_metadata_filter) 获取日志所属的k8s相关的meta信息
5. add_topic: 按照一定规则，添加kafka topic相关字段
6. filter_log: 根据正则表达式过滤日志
7. log_2_json: [fluent-plugin-json-in-json](https://github.com/gmr/fluent-plugin-json-in-json)，把业务日志转成JSON格式，方便在Kibana上查看
8. record_transformer: [fluent-plugin-record-reformer](https://github.com/sonots/fluent-plugin-record-reformer)，相关字段增加和删除

## ② ELK组件
ELK指的是Elasticsearch + Logstash + Kibana，是一套成熟的解决方案

1. Elasticsearch是一个实时的分布式搜索和分析引擎，它可以用于全文搜索，结构化搜索以及分析。它是一个建立在全文搜索引擎 Apache Lucene 基础上的搜索引擎，使用 Java 语言编写
2. Logstash 是一个实时数据处理引擎，使用 JRuby 语言编写
3. Kibana 是为 Elasticsearch 提供分析和可视化的 Web 平台，使用 JavaScript 语言编写。它可以在 Elasticsearch 的索引中查找，交互数据，并生成各种维度的表图

当前线上，Logstash定时发现Kafka是否有新的topic，然后消费这些topic，Logstash消费完成后数据写入ES，并定时创建Kibana index pattern支持用户进行日志查看。

# 五. 日志运维
## ① 稳定性
日志的稳定性由以下3个方面来保证

1. 日志支持降级: 支持根据正则表达式进行全局过滤，保证在数据量很大的情况下，Fluentd采集堆积严重后，只采集最有用的数据
2. 全链路监控
    + 采集端监控
       + Fluentd采集监控: 按各个维度监控当前Fluentd采集的日志量
       + Fluentd进程监控: Fluentd采集进程监控，是否Down，是否有大量ERROR日志
       + Fluentd资源使用监控: Fluentd Pod CPU、Mem资源使用监控
       + Fluentd Buffer堆积监控: Fluentd Buffer目录堆积监控
    + Kafka监控
       + Kafka消费监控: Kafka topic消费监控
       + Kafka实例监控: Kafka 实例监控
    + ELK监控
       + Logstash消费Kafka topic监控 
       + Logstash实例自身监控
       + Elasticsearch集群监控
3. 告警

## ② 可靠性
日志的可靠性主要体现在2个方面，日志如何保证不会丢失 以及 日志如何保证不会重复采集

1. 日志如何保证不会丢失？
    + Fluentd output插件使用file buffer，保证Kafka在不可用的时候数据可以先缓存在buffer目录下 (32G)
    + Fluentd输出到Kafka的时候，如果写失败会无限重试，保证数据不丢
    + Kafka topic多Replica，保证数据高可用
    + 数据存储到ES的时候设置多副本，保证高可用
2. 日志如何保证不会重复采集？
    + Fluentd使用in_tail插件做为输入插件，会有checkpoint机制，会持久化每个日志文件当前读取到的行号，重启之后从上次读到的地方继续读取（持久化的间隔）
    
我们会使用Fluentd buffer来做为缓冲，当后端存储挂掉的时候可以避免数据丢失

![](https://github.com/chenguolin/chenguolin.github.io/blob/master/data/image/k8s-logging-fluentd-buffer.png?raw=true)

## ③ 扩展性
日志的扩展性主要体现在4个方面

1. Fluentd采集来不及时，可以通过在容器内起更多的Fluentd进程，线性扩展Fluentd的采集性能
2. 写Kafka出现瓶颈的时候，可以通过添加Kafka broker实例，提高写能力
3. Logstash消费延迟很大，导致数据不能实时写入ES，在Kibana没办法查看到最新的日志，通过扩topic partition，提高消费并发能力，保证数据能被实时消费
4. ES存储空间不足时，可以通过扩容来解决

## ④ 易用性
在易用性上，我们主要做以下2点优化

1. 对Kibana源码进行改造，对用户展示查看更加优
2. 支持多集群，Kibana统一入口，不同集群ES索引对应同一个index pattern，对用户友好

# 六. 其它
对于那些没有办法通过K8s进行部署的业务，我们现在的部署方式采用的是通过Docker compose进行部署，使用Docker compose部署的容器有以下几个特点。

1. 容器更新的比较频繁
2. 容器的日志文件在 /var/lib/docker/containers/{container_id}/xxxx.log 目录下，并不能像K8s一样会软链接到 /var/log/containers 目录下
3. 由于Fluentd没有办法支持热更新配置文件，因此当容器不断的启动更新的时候Fluentd会需要不断变更配置文件，以支持最新容器文件的采集

我们采用的方案是对Fluentd使用Golang进行一层封装，我们命名采集器为 `log-collector`，当log-collector启动的时候会先根据当前的已有容器列表生成一序列Fluentd的配置文件，然后启动Fluentd进程进行采集，同时log-collector也监听Docker的容器事件，当有新容器生成，或者容器销毁的时候及时变更配置文件并重启Fluentd。

