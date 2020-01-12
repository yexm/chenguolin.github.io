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

我们先来创建一个 Job，yaml配置文件如下，Job 对象的yaml属性字段的设置可以参考 [kubernetes api Job](https://github.com/kubernetes/api/blob/master/batch/v1/types.go#L28)

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
   DURATION     指已经运行的时间
   ```
3. 查看Job管理Pod
   ```
   $ kubectl get pod -n kube-system | grep pi
   NAME                           READY   STATUS              RESTARTS   AGE
   pi-66drv                       0/1     Running             0          5m22s
   ```

`Job 控制器控制的对象，直接就是 Pod。Job 控制器会根据实际在 Running 状态 Pod 的数目、已经成功退出的 Pod 的数目，以及 parallelism、completions 参数的值共同计算出在这个周期里，应该创建或者删除的 Pod 数目，然后调用 Kubernetes API 来执行这个操作。`

## ② 命令
1. 创建Job: `kubectl apply -f job.yaml`
2. 列出Job: `kubectl get job -n {namespace}`
3. 删除Job: `kubectl delete job -n {namespace}`
4. 描述Job: `kubectl describe job -n {namespace}`

## ③ 属性
为了更好的了解 Job，我们需要熟悉 Jobyaml 配置文件相关属性字段的含义，我们通过下面这个例子来了解一下DaemonSet。有关 DaemonSet 属性字段的相关含义，可以参考 kubernetes api extensions/v1beta1/types DaemonSet



# 三. CronJob
