---
layout:  post  # 使用的布局（不需要改）
catalog: true  # 是否归档
author: 陈国林 # 作者
tags:          #标签
    - Kubernetes
---

# 一. 背景
前面几篇文章我们总结了 kubernetes 中 Pod、Deployment、Daemonset 等API对象的概念以及使用方法，通过 Deployment、Daemonset 的使用方法我们能知道
它们的主要应用场景是 `在线业务` 即 Long Running Task（长作业），比如我们经常使用的 Nginx、Tomcat，以及 MySQL 等等。这些应用一旦运行起来，除非出错或者停止，不然它的容器进程会一直保持在 Running 状态。

我们知道除了在线业务还有`离线业务`，所谓离线业务指的是这种应用在计算完成后就直接退出了，并不会一直处于Running状态。如果我们依然用 Deployment 来管理这种业务的话，会发现当Pod在计算完成结束后退出，但是又被 Deployment Controller 不断地重启，所以并不符合我们的预期。kubernetes 提供了 [Job](https://github.com/kubernetes/api/blob/master/batch/v1/types.go#L28) 和 [CronJob](https://github.com/kubernetes/api/blob/master/batch/v2alpha1/types.go#L58) 两种API 对象用于实现`离线业务`。

# 二. Job
## ① 介绍
`Job`用于创建一个或多个Pod，并确保这些Pod成功结束。当这些Pod都运行成功之后，Job就被标记为成功。当删除一个Job的时候，它所管理的Pod也会被一并删除。常用于离线计算场景，比如在数据分析、计算等。

我们先来创建一个 Job，yaml配置文件如下

```
apiVersion: batch/v1
kind: Job
metadata:
  name: pi
  namespace: kube-system
spec:
  template:
    spec:
      containers:
      - name: pi
        image: perl
        command: ["perl",  "-Mbignum=bpi", "-wle", "print bpi(2000)"]
      restartPolicy: Never
  backoffLimit: 4
```

1. 创建Job: `kubectl apply -f job.yaml`
2. 查看Job列表
   ```
   $ kubectl get job -n kube-system
   NAME   COMPLETIONS   DURATION   AGE
   pi     0/1           105s       105s
   
   NAME         指Job的名称
   COMPLETIONS  值已经完成的Pod数
   DURATION     指运行的时间
   AGE          指Job创建了多久
   ```
3. 查看Job管理Pod
   ```
   $ kubectl get pod -n kube-system | grep pi
   NAME                           READY   STATUS              RESTARTS   AGE
   pi-66drv                       0/1     Running             0          5m22s
   ```

`Job 控制器控制的对象，直接就是 Pod。Job 控制器会根据实际在 Running 状态 Pod 的数目、已经成功退出的 Pod 的数目，以及 parallelism、completions 参数的值共同计算出在这个周期里，应该创建或者删除的 Pod 数目，然后调用 Kubernetes API 来执行这个操作。`

Job的使用场景大部分都是 `外部控制器 + Job` 结合使用，因为kubernetes本身对任务调度做的并不是很好（DAG这种任务调度kubernetes就无能为力），所以大部分情况下是通过外部的控制器实现任务调度Pipeline，然后每个任务通过Kubernetes Job来实现。

## ② 命令
1. 创建Job: `kubectl apply -f job.yaml`
2. 列出Job: `kubectl get job -n {namespace}`
3. 删除Job: `kubectl delete job -n {namespace}`
4. 描述Job: `kubectl describe job -n {namespace}`

## ③ 属性
为了更好的了解 Job，我们需要熟悉 Job yaml 配置文件相关属性字段的含义，我们通过下面这个例子来了解一下 Job。有关 Job 属性字段的相关含义，可以参考 [kubernetes api Job](https://github.com/kubernetes/api/blob/master/batch/v1/types.go#L28)

```
apiVersion: batch/v1
kind: Job
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"batch/v1","kind":"Job","metadata":{"annotations":{},"name":"pi","namespace":"kube-system"},"spec":{"backoffLimit":4,"template":{"spec":{"containers":[{"command":["perl","-Mbignum=bpi","-wle","print bpi(2000)"],"image":"perl","name":"pi"}],"restartPolicy":"Never"}}}}
  creationTimestamp: "2020-01-12T05:35:14Z"
  labels:
    controller-uid: 8cb73b38-f811-424b-a7da-35d972cc6ec1
    job-name: pi
  name: pi
  namespace: kube-system
  resourceVersion: "1239574"
  selfLink: /apis/batch/v1/namespaces/kube-system/jobs/pi
  uid: 8cb73b38-f811-424b-a7da-35d972cc6ec1
spec:
  backoffLimit: 4
  completions: 1
  parallelism: 1
  selector:
    matchLabels:
      controller-uid: 8cb73b38-f811-424b-a7da-35d972cc6ec1
  template:
    metadata:
      creationTimestamp: null
      labels:
        controller-uid: 8cb73b38-f811-424b-a7da-35d972cc6ec1
        job-name: pi
    spec:
      containers:
      - command:
        - perl
        - -Mbignum=bpi
        - -wle
        - print bpi(2000)
        image: perl
        imagePullPolicy: Always
        name: pi
        resources: {}
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
      dnsPolicy: ClusterFirst
      restartPolicy: Never
      schedulerName: default-scheduler
      securityContext: {}
      terminationGracePeriodSeconds: 30
status:
  active: 1
  startTime: "2020-01-12T05:35:14Z"
```

### type字段
type 相关的字段的定义可以参考 [kubernetes apimachinery meta/v1/types TypeMeta](https://github.com/kubernetes/apimachinery/blob/master/pkg/apis/meta/v1/types.go#L41) 主要是以下字段

1. apiVersion: 使用的API对象的版本，可以参考kubernetes github源码 v1beta1 就是表示版本
2. kind: API对象类型，对于 Job 来说值一直是 Job

### meta字段
meta 相关的字段的定义可以参考 [kubernetes apimachinery meta/v1/types ObjectMeta](https://github.com/kubernetes/apimachinery/blob/master/pkg/apis/meta/v1/types.go#L110) 主要是以下字段

1. name: Job名称，一般由业务在yaml文件里面设置
2. namespace: Job所属的namespace （kubernetes namespace类似组的概念，和linux namespace不同）
3. creationTimestamp: Job创建时间
4. labels: Job相关的label，有些是用户设置的，有些则是kubernetes自动设置的
5. uid: kubernetes自动生成的唯一的pod id
6. selfLink: 当前Job的url，可以通过访问该url获取到Job的相关信息
7. annotations: 用户自己设置的一些key-value键值对注释，类似labels

### spec字段
spec 相关的字段的定义可以参考 [kubernetes apimachinery meta/v1/types JobSpec](https://github.com/kubernetes/api/blob/master/batch/v1/types.go#L61) 主要是以下字段

1. backoffLimit: Job 重试次数，默认为6，当重试次数耗尽的时候Job被标记为failed （采用指数退避的方式进行重试）
2. completions: Job 至少要完成的 Pod 数目（Job 的最小完成数）
3. parallelism: Job J最多可以同时启动多少个 Pod 运行 （Job 的并发运行数）
4. selector: 筛选Job要管理的Pod，kubernetes会自动在Pod template加上 controller-uid 这个label，主要是为了避免不同 Job 对象所管理的 Pod 发生重合（Job 对象并不要求你定义一个 spec.selector 来描述要管理哪些 Pod）
5. template: Pod template，用于描述Pod，具体可以参考 [kubernetes pod属性](https://chenguolin.github.io/2019/03/27/Kubernetes-17-Kubernetes%E4%B9%8BPod/#-pod%E5%B1%9E%E6%80%A7)
6. activeDeadlineSeconds: Job最长运行时间，当Job运行超过这个时间后会被强制终止，它所管理的Pod也会被强制终止
7. ttlSecondsAfterFinished: Kubernetes v1.12 提供的特性，用于描述Job的TTL。默认情况下当Job成功后，如果没有收到delete是不会被kubernetes删除的，太多的Job对象对 apiserver 性能会有影响，因此我们要保证能够自动清理掉历史的Job对象。

### status字段
status 相关的字段的定义可以参考 [kubernetes api extensions/v1beta1/types JobStatus](https://github.com/kubernetes/api/blob/master/batch/v1/types.go#L132) 主要是以下字段

1. conditions: Job的一些状态，Job就两种状态 Complete 和 Failed，可以参考[JobCondition](https://github.com/kubernetes/api/blob/master/batch/v1/types.go#L176)
2. startTime: Job运行开始时间
3. completionTime: Job运行结束时间
4. active: Job所管理的Pod中，有多少个是正在运行的
5. succeeded: Job所管理的Pod中，有多少个是运行成功的
6. failed: Job所管理的Pod中，有多少个是运行失败的

# 三. CronJob
## ① 介绍
除了 Job 之外，Kubernetes还提供了另外一个 API 对象 CronJob，顾名思义 CronJob 描述的，正是定时任务（类似Linux 系统的 Crontab）。Kubernetes Cronjob 支持定期创建 Job，通过管理 Job 达到定时创建任务的目的。

我们先来创建一个 Job，yaml配置文件如下

```
apiVersion: batch/v1beta1
kind: CronJob
metadata:
  name: hello
  namespace: kube-system
spec:
  schedule: "*/1 * * * *"
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: hello
            image: busybox
            args:
            - /bin/sh
            - -c
            - date; echo Hello from the Kubernetes cluster
          restartPolicy: OnFailure
```

1. 创建Cronjob: `kubectl apply -f cronjob.yaml`
2. 查看Cronjob列表
   ```
   $ kubectl get cronjob -n kube-system
   NAME    SCHEDULE      SUSPEND   ACTIVE   LAST SCHEDULE   AGE
   hello   */1 * * * *   False     0        30s             39s
   
   NAME      表示定时任务名称
   SCHEDULE  表示定时调度的规则 （同Unix Cron格式一样）
   SUSPEND   表示定时任务是否处于挂起状态
   ACTIVE    表示当前有多少个Job正在运行
   LAST SCHEDULE  表示上一次调度的时间（30s表示30秒之前）
   AGE       表示CronJob创建了多久
   ```
3. 查看Cronjob管理的Job
   ```
   $ kubectl get job -n kube-system
   NAME               COMPLETIONS   DURATION   AGE
   hello-1578815880   1/1           6s         3m3s
   hello-1578815940   1/1           10s        2m3s
   hello-1578816000   1/1           6s         63s
   ```
4. 查看Job管理的Pod
   ```
   $ kubectl get pod -n kube-system
   NAME                              READY   STATUS      RESTARTS   AGE
   hello-1578815940-lcbtc            0/1     Completed   0          2m42s
   hello-1578816000-29kd5            0/1     Completed   0          102s
   hello-1578816060-9c2zc            0/1     Completed   0          42s
   ```
   
由上可知，Cronjob 控制器控制的对象是 Job，Cronjob 根据业务配置的规则，定期创建 Job，Job 再去管理创建Pod。

## ② 命令
1. 创建Cronjob: `kubectl apply -f cronjob.yaml`
2. 列出Cronjob: `kubectl get cronjob -n {namespace}`
3. 删除Cronjob: `kubectl delete cronjob -n {namespace} {cronjob-name}`
4. 查看Cronjob: `kubectl describe cronjob -n {namespace} {cronjob-name}`

## ③ 属性
为了更好的了解 Cronjob，我们需要熟悉 Cronjob yaml 配置文件相关属性字段的含义，我们通过下面这个例子来了解一下 Cronjob。有关 Cronjob 属性字段的相关含义，可以参考 [kubernetes api Cronjob](https://github.com/kubernetes/api/blob/master/batch/v1beta1/types.go#L58)

```
apiVersion: batch/v1beta1
kind: CronJob
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"batch/v1beta1","kind":"CronJob","metadata":{"annotations":{},"name":"hello","namespace":"kube-system"},"spec":{"jobTemplate":{"spec":{"template":{"spec":{"containers":[{"args":["/bin/sh","-c","date; echo Hello from the Kubernetes cluster"],"image":"busybox","name":"hello"}],"restartPolicy":"OnFailure"}}}},"schedule":"*/1 * * * *"}}
  creationTimestamp: "2020-01-12T07:34:51Z"
  name: hello
  namespace: kube-system
  resourceVersion: "1262638"
  selfLink: /apis/batch/v1beta1/namespaces/kube-system/cronjobs/hello
  uid: 5eee4896-5dc9-4a76-a3e6-23b594c4067f
spec:
  concurrencyPolicy: Allow
  failedJobsHistoryLimit: 1
  jobTemplate:
    metadata:
      creationTimestamp: null
    spec:
      template:
        metadata:
          creationTimestamp: null
        spec:
          containers:
          - args:
            - /bin/sh
            - -c
            - date; echo Hello from the Kubernetes cluster
            image: busybox
            imagePullPolicy: Always
            name: hello
            resources: {}
            terminationMessagePath: /dev/termination-log
            terminationMessagePolicy: File
          dnsPolicy: ClusterFirst
          restartPolicy: OnFailure
          schedulerName: default-scheduler
          securityContext: {}
          terminationGracePeriodSeconds: 30
  schedule: '*/1 * * * *'
  successfulJobsHistoryLimit: 3
  suspend: false
status:
  lastScheduleTime: "2020-01-12T08:14:00Z"
```

### type字段
type 相关的字段的定义可以参考 [kubernetes apimachinery meta/v1/types TypeMeta](https://github.com/kubernetes/apimachinery/blob/master/pkg/apis/meta/v1/types.go#L41) 主要是以下字段

1. apiVersion: 使用的API对象的版本，可以参考kubernetes github源码 v1beta1 就是表示版本
2. kind: API对象类型，对于 Cronjob 来说值一直是 Cronjob

### meta字段
meta 相关的字段的定义可以参考 [kubernetes apimachinery meta/v1/types ObjectMeta](https://github.com/kubernetes/apimachinery/blob/master/pkg/apis/meta/v1/types.go#L110) 主要是以下字段

1. name: Cronjob 名称，一般由业务在yaml文件里面设置，`不能超过52字符，因为CronJob会自动添加11个字符生成Job名称，Job名称最多不超过63个`
2. namespace: Cronjob 所属的namespace （kubernetes namespace类似组的概念，和linux namespace不同）
3. creationTimestamp: Cronjob 创建时间
4. labels: Cronjob 相关的label，有些是用户设置的，有些则是kubernetes自动设置的
5. uid: kubernetes自动生成的唯一的pod id
6. selfLink: 当前Cronjob的url，可以通过访问该url获取到Cronjob的相关信息
7. annotations: 用户自己设置的一些key-value键值对注释，类似labels

### spec字段
spec 相关的字段的定义可以参考 [kubernetes apimachinery meta/batch/v1beta1 CronJobSpec](https://github.com/kubernetes/api/blob/master/batch/v1beta1/types.go#L92) 主要是以下字段

1. schedule: Cron的格式，可以参考 [Cron wiki](https://en.wikipedia.org/wiki/Cron)
2. concurrencyPolicy: 执行Job的策略，目前支持以下3种
    + Allow: 运行同时运行多个Job，当上一个Job还未运行结束的时候，又到了新的调度周期，那么会创建一个新的Job （这是默认的策略）
    + Forbid: 禁止同时运行多个Job，当上一个Job还未运行结束的时候，又到了新的调度周期，那么会跳过
    + Replace: 替换老的Job，当上一个Job还未运行结束的时候，又到了新的调度周期，那么会创建一个新的代替已有的Job
3. suspend: 当前Cronjob是否处于挂起状态，如果是的话则不会再创建新的Job
4. jobTemplate: job template，主要是描述Job的，可以参考 [kubernetes job spec字段](https://chenguolin.github.io/2019/03/30/Kubernetes-20-Kubernetes%E4%B9%8BCronJob%E4%B8%8EJob/#spec%E5%AD%97%E6%AE%B5)
5. successfulJobsHistoryLimit: 保留多少个成功运行的Job对象，默认是3个
6. failedJobsHistoryLimit: 保留多少个失败运行的Job对象，默认是1个

### status字段
status 相关的字段的定义可以参考 [kubernetes apimachinery meta/batch/v1beta1 CronJobStatus](https://github.com/kubernetes/api/blob/master/batch/v1beta1/types.go#L150) 主要是以下字段

1. active: 当前正在运行的Job列表
2. lastScheduleTime: 最后一次调度的时间

