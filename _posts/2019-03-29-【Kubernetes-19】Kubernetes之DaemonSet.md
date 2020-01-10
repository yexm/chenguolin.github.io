---
layout:  post  # 使用的布局（不需要改）
catalog: true  # 是否归档
author: 陈国林 # 作者
tags:          #标签
    - Kubernetes
---

# 一. 背景
上一篇文章 [kubernetes deployment](https://chenguolin.github.io/2019/03/28/Kubernetes-18-Kubernetes%E4%B9%8BDeployment/) 我们学习了 Deployment 这个API对象，是kubernetes集群中用的最多的对象类型。Deployment 支持同时运行多个Pod，以及Pod弹性伸缩，kubernetes 基于Pod粒度进行调度，对业务屏蔽掉基础环境，业务不需要关心运行在哪个节点上。

但是，有些情况我们希望能够在 kubernetes 集群的每个节点上都启动一个 Pod，做为节点的 Daemon 进程，例如以下几个例子

1. 各种网络插件的 Agent 组件，必须运行在每一个节点上，用来处理这个节点上的容器网络
2. 各种存储插件的 Agent 组件，必须运行在每一个节点上，用来在这个节点上挂载远程存储目录，操作容器的 Volume 目录
3. 各种监控组件和日志组件，也必须运行在每一个节点上，负责这个节点上的监控信息和日志采集

Kubernetes DaemonSet 正是用来解决以上问题的，通过 DaemonSet 在 Kubernetes 集群每个节点都运行一个 Daemon Pod。当集群加入一个新节点的时候，会自动创建 DaemonSet 定义的 Pod。当删除节点的时候，会自动删除 DaemonSet 定义的 Pod。

DaemonSet API 对象，它的控制器流程源码可以参考 [kubernetes controller Daemonset](https://github.com/kubernetes/kubernetes/tree/master/pkg/controller/daemon)。

# 二. Daemonset
我们先来创建一个 Daemonset，yaml配置文件如下

```
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluentd-elasticsearch
  namespace: kube-system
  labels:
    k8s-app: fluentd-logging
spec:
  selector:
    matchLabels:
      name: fluentd-elasticsearch
  template:
    metadata:
      labels:
        name: fluentd-elasticsearch
    spec:
      tolerations:
      - key: node-role.kubernetes.io/master
        effect: NoSchedule
      containers:
      - name: fluentd-elasticsearch
        image: quay.io/fluentd_elasticsearch/fluentd:v2.5.2
        resources:
          limits:
            memory: 200Mi
          requests:
            cpu: 100m
            memory: 200Mi
        volumeMounts:
        - name: varlog
          mountPath: /var/log
        - name: varlibdockercontainers
          mountPath: /var/lib/docker/containers
          readOnly: true
      terminationGracePeriodSeconds: 30
      volumes:
      - name: varlog
        hostPath:
          path: /var/log
      - name: varlibdockercontainers
        hostPath:
          path: /var/lib/docker/containers
```

1. 创建Daemonset `kubectl apply -f daemonset.yaml` （使用kube-system namespace）
2. 查看集群的状态，发现创建1个Daemonset名为 fluentd-elasticsearch 以及 3个Pod名为 fluentd-elasticsearch-xxxx
   ```
   $ kubectl get ds -n kube-system fluentd-elasticsearch
   NAME                    DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
   fluentd-elasticsearch   3         3         3       3            3           <none>          13m
   
   $ kubectl get pod -n kube-system  | grep elasticsearch
   fluentd-elasticsearch-n29vc      1/1     Running       0     15m     10.244.0.40    ecs-s6-large-2-linux-20200105130533
   fluentd-elasticsearch-n29vc      1/1     Running       0     15m     10.244.1.2     k8s-worker-node-0001
   fluentd-elasticsearch-n29vc      1/1     Running       0     15m     10.244.1.3     k8s-worker-node-0002
   ```
3. 查看当前节点，确认集群有3个node
   ```
   $ kubectl get nodes -o wide
   NAME      STATUS     ROLES    AGE    VERSION   INTERNAL-IP     EXTERNAL-IP   OS-IMAGE    KERNEL-VERSION     CONTAINER-RUNTIME
   ecs-s6-large-2-linux-20200105130533   Ready      master   4d7h   v1.17.0   192.168.0.14    <none>        CentOS Linux 7 (Core)   3.10.0-1062.9.1.el7.x86_64   docker://19.3.5
   k8s-worker-node-0001                  Ready      node     4d6h   v1.17.0   192.168.0.202   <none>        CentOS Linux 7 (Core)   3.10.0-1062.9.1.el7.x86_64   docker://19.3.5
   k8s-worker-node-0002                  Ready      node     4d6h   v1.17.0   192.168.0.239   <none>        CentOS Linux 7 (Core)   3.10.0-1062.1.1.el7.x86_64   docker://19.3.5
   ```
4. 从上面的信息可以确认每个节点都部署了一个 Pod

DaemonSet 控制器的逻辑如下，DaemonSet 控制器首先从 Etcd 里获取所有的 Node 列表，然后遍历所有的 Node，然后检查当前这个 Node 是不是有对应的 Pod 在运行了。如果没有 Pod 则在当前节点上创建一个 Pod，如果已经有 Pod 并且数量为1说明是正常的，如果已经有 Pod 但是数量有多个那要把多余的 Pod 从这个 Node 上删除掉。

# 三. 使用
## ① 常用命令
1. 创建DaemonSet: `kubectl apply -f deployment.yaml`
2. 列出DaemonSet: `kubectl get daemonset -n {namespace}`
3. 查看DaemonSet: `kubectl get daemonset -n {namespace} {daemonset-name} -o yaml`
4. 描述DaemonSet: `kubectl describe daemonset -n {namespace} {daemonset-name}`
5. 编辑DaemonSet: `kubectl edit daemonset -n {namespace} {daemonset-name}`
6. 删除DaemonSet: `kubectl delete daemonset -n {namespace} {daemonset-name}`
7. 查看滚动升级状态: `kubectl rollout status daemonset -n {namespace} {daemonset-name}`
8. 查看Daemonset历史版本: `kubectl rollout history -n {namespace} daemonset {daemonset-name}`

## ② 属性字段
为了更好的了解DaemonSet，我们需要熟悉DaemonSet yaml 配置文件相关属性字段的含义，我们通过下面这个例子来了解一下DaemonSet。有关 DaemonSet 属性字段的相关含义，可以参考 [kubernetes api extensions/v1beta1/types DaemonSet](https://github.com/kubernetes/api/blob/master/extensions/v1beta1/types.go#L482)

```
apiVersion: extensions/v1beta1
kind: DaemonSet
metadata:
  annotations:
    deprecated.daemonset.template.generation: "1"
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"apps/v1","kind":"DaemonSet","metadata":{"annotations":{},"labels":{"k8s-app":"fluentd-logging"},"name":"fluentd-elasticsearch","namespace":"kube-system"},"spec":{"selector":{"matchLabels":{"name":"fluentd-elasticsearch"}},"template":{"metadata":{"labels":{"name":"fluentd-elasticsearch"}},"spec":{"containers":[{"image":"quay.io/fluentd_elasticsearch/fluentd:v2.5.2","name":"fluentd-elasticsearch","resources":{"limits":{"memory":"200Mi"},"requests":{"cpu":"100m","memory":"200Mi"}},"volumeMounts":[{"mountPath":"/var/log","name":"varlog"},{"mountPath":"/var/lib/docker/containers","name":"varlibdockercontainers","readOnly":true}]}],"terminationGracePeriodSeconds":30,"tolerations":[{"effect":"NoSchedule","key":"node-role.kubernetes.io/master"}],"volumes":[{"hostPath":{"path":"/var/log"},"name":"varlog"},{"hostPath":{"path":"/var/lib/docker/containers"},"name":"varlibdockercontainers"}]}}}}
  creationTimestamp: "2020-01-10T09:06:32Z"
  generation: 1
  labels:
    k8s-app: fluentd-logging
  name: fluentd-elasticsearch
  namespace: kube-system
  resourceVersion: "872623"
  selfLink: /apis/apps/v1/namespaces/kube-system/daemonsets/fluentd-elasticsearch
  uid: d7366dae-e55f-466c-94a6-30a30a5127eb
spec:
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      name: fluentd-elasticsearch
  template:
    metadata:
      creationTimestamp: null
      labels:
        name: fluentd-elasticsearch
    spec:
      containers:
      - image: quay.io/fluentd_elasticsearch/fluentd:v2.5.2
        imagePullPolicy: IfNotPresent
        name: fluentd-elasticsearch
        resources:
          limits:
            memory: 200Mi
          requests:
            cpu: 100m
            memory: 200Mi
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
        volumeMounts:
        - mountPath: /var/log
          name: varlog
        - mountPath: /var/lib/docker/containers
          name: varlibdockercontainers
          readOnly: true
      dnsPolicy: ClusterFirst
      restartPolicy: Always
      schedulerName: default-scheduler
      securityContext: {}
      terminationGracePeriodSeconds: 30
      tolerations:
      - effect: NoSchedule
        key: node-role.kubernetes.io/master
      volumes:
      - hostPath:
          path: /var/log
          type: ""
        name: varlog
      - hostPath:
          path: /var/lib/docker/containers
          type: ""
        name: varlibdockercontainers
  updateStrategy:
    rollingUpdate:
      maxUnavailable: 1
    type: RollingUpdate
status:
  currentNumberScheduled: 3
  desiredNumberScheduled: 3
  numberAvailable: 3
  numberMisscheduled: 0
  numberReady: 3
  observedGeneration: 1
  updatedNumberScheduled: 3
```

### type字段
type 相关的字段的定义可以参考 [kubernetes apimachinery meta/v1/types TypeMeta](https://github.com/kubernetes/apimachinery/blob/master/pkg/apis/meta/v1/types.go#L41) 主要是以下字段

1. apiVersion: 使用的API对象的版本，可以参考kubernetes github源码 v1beta1 就是表示版本
2. kind: API对象类型，对于 Daemonset 来说值一直是 DaemonSet

### meta字段
meta 相关的字段的定义可以参考 [kubernetes apimachinery meta/v1/types ObjectMeta](https://github.com/kubernetes/apimachinery/blob/master/pkg/apis/meta/v1/types.go#L110) 主要是以下字段

1. name: Daemonset名称，一般由业务在yaml文件里面设置
2. namespace: Daemonset所属的namespace （kubernetes namespace类似组的概念，和linux namespace不同）
3. creationTimestamp: Daemonset创建时间
4. labels: Daemonset相关的label，有些是用户设置的，有些则是kubernetes自动设置的
5. uid: kubernetes自动生成的唯一的pod id
6. selfLink: 当前Daemonset的url，可以通过访问该url获取到Daemonset的相关信息
7. annotations: 用户自己设置的一些key-value键值对注释，类似labels

### spec字段
spec 相关的字段的定义可以参考 [kubernetes apimachinery meta/v1/types DaemonSetSpec](https://github.com/kubernetes/api/blob/master/extensions/v1beta1/types.go#L362) 主要是以下字段

1. revisionHistoryLimit: Daemonset保留几个版本
2. selector: 筛选Daemonset要管理的Pod，一般和template内metadata.labels保持一致
3. template: Pod template，用于描述Pod，具体可以参考 [kubernetes pod属性](https://chenguolin.github.io/2019/03/27/Kubernetes-17-Kubernetes%E4%B9%8BPod/#-pod%E5%B1%9E%E6%80%A7)
4. updateStrategy: Daemonset更新策略，目前支持 `RollingUpdate`和`OnDelete`

### status字段
status 相关的字段的定义可以参考 [kubernetes api extensions/v1beta1/types DaemonSetStatus](https://github.com/kubernetes/api/blob/master/extensions/v1beta1/types.go#L402) 主要是以下字段

1. currentNumberScheduled: 已经在多少个节点上运行 Pod
2. desiredNumberScheduled: 期望在多少个节点运行 Pod
3. numberAvailable: 多少个节点上 Pod 可用（能对外提供服务）
4. numberMisscheduled: 多少个节点上未运行 Pod
5. numberReady: 多少个节点上 Pod ready （一般等价于numberMisscheduled）
6. observedGeneration: 版本
7. updatedNumberScheduled: 多少个节点已经更新到最新 Pod

## ③ 滚动升级




