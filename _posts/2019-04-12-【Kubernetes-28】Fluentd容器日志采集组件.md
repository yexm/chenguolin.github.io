---
layout:  post  # 使用的布局（不需要改）
catalog: true  # 是否归档
author: 陈国林 # 作者
tags:          #标签
    - Kubernetes
---

# 一. 概述
上一篇文章我们整体介绍了 [容器日志采集方案](https://chenguolin.github.io/2019/06/19/K8s-6-%E5%AE%B9%E5%99%A8%E6%97%A5%E5%BF%97%E9%87%87%E9%9B%86%E6%96%B9%E6%A1%88/)，我们知道K8s集群的日志采集组件用的最多的是 `Fluentd`。关于 Fluentd 的详细介绍，可以参考以下文章。这篇文章，我会总结一下在K8s集群使用 Fluentd 进行容器日志采集的实现方案，以及相关的配置。

1. [Fluentd基础介绍](https://chenguolin.github.io/2019/02/26/Fluentd-1-Fluentd%E5%9F%BA%E7%A1%80%E4%BB%8B%E7%BB%8D/)
2. [Fluentd命令介绍](https://chenguolin.github.io/2019/02/27/Fluentd-2-Fluentd%E5%91%BD%E4%BB%A4%E4%BB%8B%E7%BB%8D/)
3. [Fluentd插件介绍](https://chenguolin.github.io/2019/02/28/Fluentd-3-Fluentd%E6%8F%92%E4%BB%B6%E4%BB%8B%E7%BB%8D/)
4. [Fluentd使用举例](https://chenguolin.github.io/2019/03/01/Fluentd-4-Fluentd%E4%BD%BF%E7%94%A8%E4%B8%BE%E4%BE%8B/)
5. [Fluentd插件开发](https://chenguolin.github.io/2019/03/02/Fluentd-5-Fluentd%E6%8F%92%E4%BB%B6%E5%BC%80%E5%8F%91/)
6. [Fluentd Prometheus监控](https://chenguolin.github.io/2019/03/05/Fluentd-6-Fluentd-Prometheus%E7%9B%91%E6%8E%A7/)
7. [Fluentd日志采集全链路方案](https://chenguolin.github.io/2019/03/07/Fluentd-7-Fluentd%E6%97%A5%E5%BF%97%E9%87%87%E9%9B%86%E5%85%A8%E9%93%BE%E8%B7%AF%E6%96%B9%E6%A1%88/)

# 二. 实现
Fluentd 整体的部署架构如下图所示，总体来说分为2部分。fluentd 相关的配置通过k8s configmap进行配置存储在 `master etcd`，fluentd 自身通过k8s daemonset 部署在每个worker节点上。采集完数据后实时输出到 Kafka，下游通过 ELK 进行处理、存储和数据查询。

![](https://github.com/chenguolin/chenguolin.github.io/blob/master/data/image/k8s-fluentd-agent.png?raw=true)

整体项目可以参考 [k8s-fluentd](https://github.com/chenguolin/k8s-fluentd)，相关的插件都是开源已有的，我们做了些定制改造。

## ① configmap
fluentd 启动需要一个 `fluent.conf` 配置文件，我们采用的方式是把 fluent.conf 文件通过K8s configmap的方式进行配置，configmap的内容大致如下。
因此，在部署fluentd daemonset之前我们需要先把更新configmap，采用 `kubectl apply -f fluentd-configmap.yml` 命令即可。

[fluentd-configmap.yml](https://github.com/chenguolin/k8s-fluentd/blob/master/yaml/k8s-fluentd-configmap.yaml)

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: fluentd-config.d-v0.0.1
  namespace: kube-system
  labels:
    k8s-app: fluentd-config
data:
  containers.input.conf: |-
    # 这里不能使用 ** 做为匹配pattern，因为detect_exceptions这个插件会把数据重新扔回input
    # 如果使用 ** 会导致input的统计数据double
    <filter raw.**>
       @type prometheus
       <metric>
         name fluentd_input_records
         type counter
         desc The total number of incoming records
       </metric>
    </filter>

    <match fluent.**>
       @type null
    </match>

    <match **fluentd**>
       @type null
    </match>

    <match raw.**>
       @type detect_exceptions
       remove_tag_prefix raw
       languages java,phpslow
       message log
       stream stream
       multiline_flush_interval 0.1
       max_bytes 500000
       max_lines 1000
    </match>

    <filter kubernetes.**>
       @type concatdocker
       key log
       stream_identity_key stream
       use_first_timestamp true
       separator ""
       multiline_end_regexp /\n$/
    </filter>

    <filter kubernetes.**>
       @type prune_log
       max_raw_log_bytes 1M
       prefix_reserve_bytes 1K
       suffix_reserve_bytes 1K
    </filter>

    <filter kubernetes.**>
       @type kubernetes_metadata
       cache_size 5000
       cache_ttl 3600
       annotation_match ["^log-.*"]
       merge_json_log false
    </filter>

    <filter kubernetes.**>
       @type addtopic
       topic_prefix "#{ENV['KAFKA_TOPIC_PREFIX']}"
    </filter>

    <filter kubernetes.**>
       @type json_in_json
       convert_non_json_log true
    </filter>

---
apiVersion: v1
kind: ConfigMap
metadata:
  name: fluentd-config-v0.0.1
  namespace: kube-system
  labels:
    k8s-app: fluentd-config
data:
  fluent.conf: |-
    <source>
       @type multiprocess
       <process>
          cmdline -c /etc/fluent/config/fluent-commons.conf -p /etc/fluent/plugins -q -o /tmp/fluent-commons.log
          sleep_before_start 1s
          sleep_before_shutdown 5s
       </process>
       <process>
          cmdline -c /etc/fluent/config/fluent-containers-0-3.conf -p /etc/fluent/plugins -q -o /tmp/fluent-containers-0-3.log
          sleep_before_start 1s
          sleep_before_shutdown 5s
       </process>
       <process>
          cmdline -c /etc/fluent/config/fluent-containers-4-7.conf -p /etc/fluent/plugins -q -o /tmp/fluent-containers-4-7.log
          sleep_before_start 1s
          sleep_before_shutdown 5s
       </process>
       <process>
          cmdline -c /etc/fluent/config/fluent-containers-8-b.conf -p /etc/fluent/plugins -q -o /tmp/fluent-containers-8-b.log
          sleep_before_start 1s
          sleep_before_shutdown 5s
       </process>
       <process>
          cmdline -c /etc/fluent/config/fluent-containers-c-f.conf -p /etc/fluent/plugins -q -o /tmp/fluent-containers-c-f.log
          sleep_before_start 1s
          sleep_before_shutdown 5s
       </process>
       <process>
          cmdline -c /etc/fluent/config/fluent-itself.conf -p /etc/fluent/plugins -q
          sleep_before_start 1s
          sleep_before_shutdown 5s
       </process>
    </source>


  fluent-commons.conf: |-
    <match fluent.**>
       @type null
    </match>

    <match **fluentd**>
       @type null
    </match>

    <source>
       @type tail
       tag kubernetes_core
       path /var/log/calico/*/current
       pos_file /var/log/fluentd-calico.log.pos
       read_from_head true
       format none
    </source>

    <source>
       @type systemd
       tag kubernetes_core
       path /var/log/journal
       filters [{ "_SYSTEMD_UNIT": ["kubelet.service", "etcd.service", "docker.service"]}]
       read_from_head true
       pos_file /var/log/fluentd-systemd.log.pos
    </source>

    <filter **>
       @type prometheus
       <metric>
          name fluentd_input_records
          type counter
          desc The total number of incoming records
       </metric>
    </filter>

    <match **>
       @type copy_ex
       <store ignore_error>
          @type kafka_buffered

          brokers "#{ENV['FLUENT_KAFKA_BROKERS']}"
          default_topic k8s_unexpected-logs
          get_kafka_client_log true

          output_include_tag false
          output_include_time false
          exclude_topic_key false
          exclude_partition_key true

          output_data_type json

          buffer_type file
          buffer_path /var/log/td-agent/buffer/td
          buffer_chunk_limit 64M
          buffer_queue_limit 512
          flush_interval 3s

          kafka_agg_max_bytes 3M
          max_send_limit_bytes 3M
          compression_codec gzip
          required_acks 1
          num_threads 4
          max_send_retries 5
          discard_kafka_delivery_failed true
          disable_retry_limit false
          max_retry_wait 1
       </store>
       <store ignore_error>
          @type mt_flowcounter
       </store>
    </match>

    # input plugin that is required to expose metrics by other prometheus
    # plugins, such as the prometheus_monitor input below.
    <source>
       @type prometheus
       bind 0.0.0.0
       port 24234
       metrics_path /metrics
    </source>

    # input plugin that collects metrics from MonitorAgent and exposes them
    # as prometheus metrics
    <source>
       @type prometheus_monitor
       # update the metrics every 5 seconds
       interval 5
       <labels>
          name k8s_core
       </labels>
    </source>

    <source>
       @type prometheus_output_monitor
       interval 5
       <labels>
          name k8s_core
       </labels>
    </source>


  fluent-containers-0-3.conf: |-
    <source>
       @type tail
       tag raw.kubernetes.*
       path /var/log/containers/*0.log,/var/log/containers/*1.log,/var/log/containers/*2.log,/var/log/containers/*3.log
       pos_file /var/log/fluentd-containers0-3.log.pos
       format json
       time_key not-parse-time
       read_from_head true
       refresh_interval 3
       enable_watch_timer true
    </source>

    @include /etc/fluent/config.d/*.conf

    <match **>
       @type copy_ex
       <store ignore_error>
          @type kafka_buffered

          brokers "#{ENV['FLUENT_KAFKA_BROKERS']}"
          default_topic k8s_unexpected-logs
          get_kafka_client_log true

          output_include_tag false
          output_include_time false
          exclude_topic_key false
          exclude_partition_key true

          output_data_type json

          buffer_type file
          buffer_path /var/log/td-agent/buffer/td0-3
          buffer_chunk_limit 64M
          buffer_queue_limit 512
          flush_interval 3s

          kafka_agg_max_bytes 3M
          max_send_limit_bytes 3M
          compression_codec gzip
          required_acks 1
          num_threads 4
          max_send_retries 5
          discard_kafka_delivery_failed true
          disable_retry_limit false
          max_retry_wait 1
       </store>
       <store ignore_error>
          @type mt_flowcounter
       </store>
    </match>

    # input plugin that is required to expose metrics by other prometheus
    # plugins, such as the prometheus_monitor input below.
    <source>
       @type prometheus
       bind 0.0.0.0
       port 24230
       metrics_path /metrics
    </source>

    # input plugin that collects metrics from MonitorAgent and exposes them
    # as prometheus metrics
    <source>
       @type prometheus_monitor
       # update the metrics every 5 seconds
       interval 5
       <labels>
          name log0_3
       </labels>
    </source>

    <source>
       @type prometheus_output_monitor
       interval 5
       <labels>
          name log0_3
       </labels>
    </source>


  fluent-containers-4-7.conf: |-
    <source>
       @type tail
       tag raw.kubernetes.*
       path /var/log/containers/*4.log,/var/log/containers/*5.log,/var/log/containers/*6.log,/var/log/containers/*7.log
       pos_file /var/log/fluentd-containers4-7.log.pos
       format json
       time_key not-parse-time
       read_from_head true
       refresh_interval 3
       enable_watch_timer true
    </source>

    @include /etc/fluent/config.d/*.conf

    <match **>
       @type copy_ex
       <store ignore_error>
          @type kafka_buffered

          brokers "#{ENV['FLUENT_KAFKA_BROKERS']}"
          default_topic k8s_unexpected-logs
          get_kafka_client_log true

          output_include_tag false
          output_include_time false
          exclude_topic_key false
          exclude_partition_key true

          output_data_type json

          buffer_type file
          buffer_path /var/log/td-agent/buffer/td4-7
          buffer_chunk_limit 64M
          buffer_queue_limit 512
          flush_interval 3s

          kafka_agg_max_bytes 3M
          max_send_limit_bytes 3M
          compression_codec gzip
          required_acks 1
          num_threads 4
          max_send_retries 5
          discard_kafka_delivery_failed true
          disable_retry_limit false
          max_retry_wait 1
       </store>
       <store ignore_error>
          @type mt_flowcounter
       </store>
    </match>

    # input plugin that is required to expose metrics by other prometheus
    # plugins, such as the prometheus_monitor input below.
    <source>
       @type prometheus
       bind 0.0.0.0
       port 24231
       metrics_path /metrics
    </source>

    # input plugin that collects metrics from MonitorAgent and exposes them
    # as prometheus metrics
    <source>
       @type prometheus_monitor
       # update the metrics every 5 seconds
       interval 5
       <labels>
          name log4_7
       </labels>
    </source>

    <source>
       @type prometheus_output_monitor
       interval 5
       <labels>
          name log4_7
       </labels>
    </source>

  fluent-containers-8-b.conf: |-
    <source>
       @type tail
       tag raw.kubernetes.*
       path /var/log/containers/*8.log,/var/log/containers/*9.log,/var/log/containers/*a.log,/var/log/containers/*b.log
       pos_file /var/log/fluentd-containers8-b.log.pos
       format json
       time_key not-parse-time
       read_from_head true
       refresh_interval 3
       enable_watch_timer true
    </source>

    @include /etc/fluent/config.d/*.conf

    <match **>
       @type copy_ex
       <store ignore_error>
          @type kafka_buffered

          brokers "#{ENV['FLUENT_KAFKA_BROKERS']}"
          default_topic k8s_unexpected-logs
          get_kafka_client_log true

          output_include_tag false
          output_include_time false
          exclude_topic_key false
          exclude_partition_key true

          output_data_type json

          buffer_type file
          buffer_path /var/log/td-agent/buffer/td8-b
          buffer_chunk_limit 64M
          buffer_queue_limit 512
          flush_interval 3s

          kafka_agg_max_bytes 3M
          max_send_limit_bytes 3M
          compression_codec gzip
          required_acks 1
          num_threads 4
          max_send_retries 5
          discard_kafka_delivery_failed true
          disable_retry_limit false
          max_retry_wait 1
       </store>
       <store ignore_error>
          @type mt_flowcounter
       </store>
    </match>

    # input plugin that is required to expose metrics by other prometheus
    # plugins, such as the prometheus_monitor input below.
    <source>
       @type prometheus
       bind 0.0.0.0
       port 24232
       metrics_path /metrics
    </source>

    # input plugin that collects metrics from MonitorAgent and exposes them
    # as prometheus metrics
    <source>
       @type prometheus_monitor
       # update the metrics every 5 seconds
       interval 5
       <labels>
          name log8_b
       </labels>
    </source>

    <source>
       @type prometheus_output_monitor
       interval 5
       <labels>
          name log8_b
       </labels>
    </source>

  fluent-containers-c-f.conf: |-
    <source>
       @type tail
       tag raw.kubernetes.*
       path /var/log/containers/*c.log,/var/log/containers/*d.log,/var/log/containers/*e.log,/var/log/containers/*f.log
       pos_file /var/log/fluentd-containersc-f.log.pos
       format json
       time_key not-parse-time
       read_from_head true
       refresh_interval 3
       enable_watch_timer true
    </source>

    @include /etc/fluent/config.d/*.conf

    <match **>
       @type copy_ex
       <store ignore_error>
          @type kafka_buffered

          brokers "#{ENV['FLUENT_KAFKA_BROKERS']}"
          default_topic k8s_unexpected-logs
          get_kafka_client_log true

          output_include_tag false
          output_include_time false
          exclude_topic_key false
          exclude_partition_key true

          output_data_type json

          buffer_type file
          buffer_path /var/log/td-agent/buffer/tdc-f
          buffer_chunk_limit 64M
          buffer_queue_limit 512
          flush_interval 3s

          kafka_agg_max_bytes 3M
          max_send_limit_bytes 3M
          compression_codec gzip
          required_acks 1
          num_threads 4
          max_send_retries 5
          discard_kafka_delivery_failed true
          disable_retry_limit false
          max_retry_wait 1
       </store>
       <store ignore_error>
          @type mt_flowcounter
       </store>
    </match>

    # input plugin that is required to expose metrics by other prometheus
    # plugins, such as the prometheus_monitor input below.
    <source>
       @type prometheus
       bind 0.0.0.0
       port 24233
       metrics_path /metrics
    </source>

    # input plugin that collects metrics from MonitorAgent and exposes them
    # as prometheus metrics
    <source>
       @type prometheus_monitor
       # update the metrics every 5 seconds
       interval 5
       <labels>
          name logc_f
       </labels>
    </source>

    <source>
       @type prometheus_output_monitor
       interval 5
       <labels>
          name logc_f
       </labels>
    </source>


  fluent-itself.conf: |-
    <source>
       @type tail
       tag raw.fluent.*
       path /tmp/fluent*.log
       pos_file /var/log/fluentd-itself.log.pos
       read_from_head true
       format none
       refresh_interval 3
       enable_watch_timer true
    </source>

    <match **>
      @type copy_ex
      <store ignore_error>
        @type rawstdout
      </store>

      <store ignore_error>
        @type fluentd_monitor
      </store>
    </match>

    # input plugin that is required to expose metrics by other prometheus
    # plugins, such as the prometheus_monitor input below.
    <source>
       @type prometheus
       bind 0.0.0.0
       port 24235
       metrics_path /metrics
    </source>

    # input plugin that collects metrics from MonitorAgent and exposes them
    # as prometheus metrics
    <source>
       @type prometheus_monitor
       # update the metrics every 5 seconds
       interval 5
       <labels>
          name fluent_log
       </labels>
    </source>

    <source>
       @type prometheus_output_monitor
       interval 5
       <labels>
          name fluent_log
       </labels>
    </source>
```

## ② daemonset
fluentd 是通过K8s daemonset的方式进行部署的，通过在每个节点部署一个fluentd pod进行容器日志采集，采集完成之后输出到Kafka topic。根据性能测试的数据，fluentd 单进程的采集性能在 `10MB ~ 20MB`，加入output Kafka之后性能会更低一些。为了支撑更高的采集性能，我们是在一个 Pod 内启动多个 fluentd 进程，每个fluentd进程采集一部分的日志，通过线性扩展的方式提升整体的采集性能，目前单个 Pod 能够支撑的采集性能是 `40MB ~ 80MB`，完全足够。

fluentd daemonset的yaml配置参考如下 [k8s-fluentd-daemonset.yaml](https://github.com/chenguolin/k8s-fluentd/blob/master/yaml/k8s-fluentd-daemonset.yaml)，部署的命令为 `kubectl apply -f k8s-fluentd-daemonset.yaml`

```
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: fluentd
  namespace: kube-system

---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRole
metadata:
  name: fluentd
  namespace: kube-system
rules:
- apiGroups:
  - ""
  resources:
  - pods
  - namespaces
  verbs:
  - get
  - list
  - watch

---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: fluentd
roleRef:
  kind: ClusterRole
  name: fluentd
  apiGroup: rbac.authorization.k8s.io
subjects:
- kind: ServiceAccount
  name: fluentd
  namespace: kube-system

---
apiVersion: extensions/v1beta1
kind: DaemonSet
metadata:
  name: fluentd
  namespace: kube-system
  labels:
    k8s-app: fluentd-logging
    kubernetes.io/cluster-service: "true"
spec:
  template:
    metadata:
      labels:
        k8s-app: fluentd-logging
        kubernetes.io/cluster-service: "true"
      annotations:
        prometheus_io_scrape: "true"
    spec:
      serviceAccount: fluentd
      serviceAccountName: fluentd
      tolerations:
      - operator: "Exists"
      containers:
      - name: fluentd
        image: k8s-fluentd:v0.0.1
        imagePullPolicy: Always
        args: [
          "/run.sh",
          "-c", "/etc/fluent/config/fluent.conf",
          "--no-supervisor", "-q",
        ]
        env:
          - name: FLUENT_KAFKA_BROKERS 
            value: "192.168.0.1:9092"
          - name: KAFKA_TOPIC_PREFIX
            value: "k8s-fluentd-"
          - name: NODE_NAME
            valueFrom:
              fieldRef:
                fieldPath: spec.nodeName
          - name: POD_NAME
            valueFrom:
              fieldRef:
                fieldPath: metadata.name
        resources:
          limits:
            cpu: 1000m
            memory: 2000Mi
          requests:
            cpu: 100m
            memory: 800Mi
        ports:
        - name: prom-0-3
          containerPort: 24230
          protocol: TCP
        - name: prom-4-7
          containerPort: 24231
          protocol: TCP
        - name: prom-8-b
          containerPort: 24232
          protocol: TCP
        - name: prom-c-f
          containerPort: 24233
          protocol: TCP
        - name: prom
          containerPort: 24234
          protocol: TCP
        - name: prom-fluent
          containerPort: 24235
          protocol: TCP
        volumeMounts:
        - name: varlog
          mountPath: /var/log
        - name: varlibdockercontainers
          mountPath: /var/lib/docker/containers
          readOnly: true
        - name: config-volume
          mountPath: /etc/fluent/config
        - name: config-d-volume
          mountPath: /etc/fluent/config.d
      imagePullSecrets:
        - name: default-secret
      terminationGracePeriodSeconds: 30
      volumes:
      - name: varlog
        hostPath:
          path: /var/log
      - name: varlibdockercontainers
        hostPath:
          path: /var/lib/docker/containers
      - name: config-volume
        configMap:
          name: fluentd-config-v0.0.1
      - name: config-d-volume
        configMap:
          name: fluentd-config.d-v0.0.1
```

# 三. 监控报警
Fluentd 相关监控报警采用的 Prometheus + Grafana 的方案，具体可以参考 [prometheus+grafana监控fluentd容器日志采集](https://chenguolin.github.io/2019/02/18/Prometheus-4-Prometheus+Grafana%E7%9B%91%E6%8E%A7Fluentd%E5%AE%B9%E5%99%A8%E6%97%A5%E5%BF%97%E9%87%87%E9%9B%86/)

