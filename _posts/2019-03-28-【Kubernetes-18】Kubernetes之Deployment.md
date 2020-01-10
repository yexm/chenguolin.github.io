---
layout:  post  # 使用的布局（不需要改）
catalog: true  # 是否归档
author: 陈国林 # 作者
tags:          #标签
    - Kubernetes
---

# 一. 背景
上一篇文章 [kubernetes之Pod](https://chenguolin.github.io/2019/03/27/Kubernetes-17-Kubernetes%E4%B9%8BPod/) 我们了解了 `Pod 这个API 对象，实际上是对容器的进一步抽象和封装而已`。Pod 对象，其实就是容器的升级版。它对容器进行了组合，添加了更多的属性和字段。这篇文章我们来了解一下 kubernetes 的 Deployment 这个API对象。

我们先来回答一个问题 `有了Pod，为什么我们还需要Deployment？`

我们知道生产环境上大部分的业务都是需要高可用的，通常实现高可用的方案是通过多副本机制。也就是说，如果我们现在有一个业务通过Pod部署，要实现高可用的话，我们就必须通过保证集群有多个相同的Pod，但是通过上一篇文章 [kubernetes之Pod](https://chenguolin.github.io/2019/03/27/Kubernetes-17-Kubernetes%E4%B9%8BPod/) 我们发现并没有一个属性字段允许同时运行多个Pod。因此，我们需要通过额外的手段来保证集群有多个Pod，这不仅给业务引入了复杂性，也这不符合kubernetes的设计原则。

因此，kubernetes 提供了多副本控制机制，常见的有 ReplicationController、ReplicaSet、Deployment、DaemonSet、Job 等。如果使用 Deployment 的话，那么业务便可以快速实现高可用，同时部署多个相同的Pod，还允许弹性伸缩。

我们在 [kubernetes架构](https://chenguolin.github.io/2019/03/24/Kubernetes-13-Kubernetes%E5%9F%BA%E7%A1%80%E4%BB%8B%E7%BB%8D/#%E4%B8%89-k8s%E6%9E%B6%E6%9E%84) 文章里提到 master 里面有一个 [kube-controller-manager](https://github.com/kubernetes/kubernetes/tree/master/cmd/kube-controller-manager)，简单的说 kube-controller-manager 用来编排某种 API 对象的。ReplicationController、ReplicaSet、Deployment、DaemonSet、Job 等对象都是通过 kube-controller-manager 内部的控制器负责的。

Deployment API 对象，它的控制器流程源码可以参考 [kubernetes controller deployment](https://github.com/kubernetes/kubernetes/tree/master/pkg/controller/deployment)。像 Deployment 这种控制器的设计原理，是 `用一种对象管理另一种对象`的艺术，控制器对象本身负责定义被管理对象的期望状态。

实际上，所有控制器最后的执行结果，要么是创建、更新一些 Pod（或者其他的 API 对象、资源），要么就是删除一些已经存在的 Pod（或者其他的 API 对象、资源）。

# 二. Deployment
Deployment 看似简单，但实际上它实现了 kubernetes 项目中一个非常重要的功能 `Pod 的水平扩展/收缩（horizontal scaling out/in）`。这个功能，是从 PaaS 时代开始，一个平台级项目就必须具备的编排能力。Deployment 实现上是通过控制 ReplicaSet 对象，然后通过 ReplicaSet 对象达到控制 Pod 的目的。

我们先来创建一个Deployment，yaml配置文件如下。当我们提交了一个 Deployment 对象后，Deployment Controller 就会立即创建一个 Pod 副本个数为 3 的 ReplicaSet。这个 ReplicaSet 的名字，则是由 Deployment 的名字和一个随机字符串共同组成。

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.7.9
        ports:
        - containerPort: 80
```

1. 创建Deployment `kubectl apply -f deployment.yaml`  （使用默认namespace）
2. 查看集群状态，发现创建1个Deployment名为 nginx-deployment，1个ReplicaSet名为nginx-deployment-54f57cf6bf 以及 3个Pod名为nginx-deployment-54f57cf6bf-xxxx
   ```
   $ kubectl get all | grep nginx
   pod/nginx-deployment-54f57cf6bf-6775m   1/1     Running   0          76s
   pod/nginx-deployment-54f57cf6bf-6wngr   1/1     Running   0          76s
   pod/nginx-deployment-54f57cf6bf-bhjzw   1/1     Running   0          76s
   deployment.apps/nginx-deployment   3/3     3            3           76s
   replicaset.apps/nginx-deployment-54f57cf6bf   3         3         3       76s
   ```
3. 我们可以确认，Deployment实现逻辑为，先创建ReplicaSet，通过ReplicaSet创建Pod
4. 查看Deployment
   ```
   $ kubectl get deploy nginx-deployment
   NAME               READY   UP-TO-DATE   AVAILABLE   AGE
   nginx-deployment   3/3     3            3           5m7s
   
   NAME        Deployment的名称，通常是业务在yaml文件里面配置的（metadata.name字段）
   READY       指当前已经处于Ready的 Pod 个数 （Ready 指的是健康检查正确）
   UP-TO-DATE  指当前处于最新版本的 Pod 个数（所谓最新版本指的是 Pod 的 Spec 部分与 Deployment 里 Pod 模板里定义的完全一致）
   AVAILABLE   指当前可用的Pod个数 （既是 Running 状态，又是最新版本，并且已经处于 Ready 状态的 Pod 的个数）
   ```
5. 查看ReplicaSet
   ```
   $ kubectl get replicaset
   NAME                          DESIRED   CURRENT   READY   AGE
   nginx-deployment-54f57cf6bf   3         3         3       11m
   
   NAME        ReplicaSet的名称，格式为 nginx-deployment-{randstring}（randstring由kubernetes根据pod-template-hash生成）
   DESIRED     指用户期望的 Pod 个数，通常是业务在yaml文件里面配置的（spec.replicas字段）
   CURRENT     指当前处于 Running 状态的 Pod 的个数
   READY       指当前已经处于Ready的 Pod 个数 （Ready 指的是健康检查正确）
   ```
6. 查看Pod
   ```
   $ kubectl get pod
   NAME                                READY   STATUS    RESTARTS   AGE
   nginx-deployment-54f57cf6bf-6775m   1/1     Running   0          14m
   nginx-deployment-54f57cf6bf-6wngr   1/1     Running   0          14m
   nginx-deployment-54f57cf6bf-bhjzw   1/1     Running   0          14m
   
   NAME        Pod的名称，格式为 nginx-deployment-54f57cf6bf-{another random string}
   READY       指当前已经处于Ready的 Pod 个数 （Ready 指的是健康检查正确）
   STATUS      指Pod的状态
   RESTARTS    指Pod重启的次数
   ```
   
`Deployment 实际上是一个两层控制器。首先，它通过 ReplicaSet 的个数来描述应用的版本。然后，它再通过 ReplicaSet 的属性（比如 replicas 的值），来保证 Pod 的副本数量。Deployment 控制 ReplicaSet（版本），ReplicaSet 控制 Pod（副本数）。`

# 三. 使用
## ① 常用命令
1. 创建Deployment: `kubectl apply -f deployment.yaml`
2. 列出Deployment: `kubectl get deployment -n {kube-system}`
3. 查看Deployment: `kubectl get deployment -n {kube-system} {deployment-name} -o yaml`
4. 描述Deployment: `kubectl describe deployment -n {kube-system} {deployment-name}`
5. 编辑Deployment: `kubectl edit deployment -n {kube-system} {deployment-name}`
6. 删除Deployment: `kubectl delete deployment -n {kube-system} {deployment-name}`
7. 查看滚动升级状态: `kubectl rollout status deployment -n {kube-system} {deployment-name}`

## ② 属性字段
为了更好的了解Deployment，我们需要熟悉Deployment yaml 配置文件相关属性字段的含义，我们通过 下面这个例子来了解一下Deployment。有关 Deployment 属性字段的相关含义，可以参考 [kubernetes api extensions/v1beta1/types Deployment](https://github.com/kubernetes/api/blob/master/extensions/v1beta1/types.go#L78)

```
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  annotations:
    deployment.kubernetes.io/revision: "1"
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"apps/v1","kind":"Deployment","metadata":{"annotations":{},"labels":{"k8s-app":"kubernetes-dashboard"},"name":"kubernetes-dashboard","namespace":"kube-system"},"spec":{"replicas":1,"revisionHistoryLimit":10,"selector":{"matchLabels":{"k8s-app":"kubernetes-dashboard"}},"template":{"metadata":{"labels":{"k8s-app":"kubernetes-dashboard"}},"spec":{"containers":[{"args":["--auto-generate-certificates"],"image":"k8s.gcr.io/kubernetes-dashboard-amd64:v1.10.1","livenessProbe":{"httpGet":{"path":"/","port":8443,"scheme":"HTTPS"},"initialDelaySeconds":30,"timeoutSeconds":30},"name":"kubernetes-dashboard","ports":[{"containerPort":8443,"protocol":"TCP"}],"volumeMounts":[{"mountPath":"/certs","name":"kubernetes-dashboard-certs"},{"mountPath":"/tmp","name":"tmp-volume"}]}],"serviceAccountName":"kubernetes-dashboard","tolerations":[{"effect":"NoSchedule","key":"node-role.kubernetes.io/master"}],"volumes":[{"name":"kubernetes-dashboard-certs","secret":{"secretName":"kubernetes-dashboard-certs"}},{"emptyDir":{},"name":"tmp-volume"}]}}}}
  creationTimestamp: "2020-01-06T06:21:35Z"
  generation: 1
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard
  namespace: kube-system
  resourceVersion: "37592"
  selfLink: /apis/apps/v1/namespaces/kube-system/deployments/kubernetes-dashboard
  uid: e48eb030-1814-4ae9-add4-df89f26ff201
spec:
  progressDeadlineSeconds: 600
  replicas: 1
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      k8s-app: kubernetes-dashboard
  strategy:
    rollingUpdate:
      maxSurge: 25%
      maxUnavailable: 25%
    type: RollingUpdate
  template:
    metadata:
      creationTimestamp: null
      labels:
        k8s-app: kubernetes-dashboard
    spec:
      containers:
      - args:
        - --auto-generate-certificates
        image: k8s.gcr.io/kubernetes-dashboard-amd64:v1.10.1
        imagePullPolicy: IfNotPresent
        livenessProbe:
          failureThreshold: 3
          httpGet:
            path: /
            port: 8443
            scheme: HTTPS
          initialDelaySeconds: 30
          periodSeconds: 10
          successThreshold: 1
          timeoutSeconds: 30
        name: kubernetes-dashboard
        ports:
        - containerPort: 8443
          protocol: TCP
        resources: {}
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
        volumeMounts:
        - mountPath: /certs
          name: kubernetes-dashboard-certs
        - mountPath: /tmp
          name: tmp-volume
      dnsPolicy: ClusterFirst
      restartPolicy: Always
      schedulerName: default-scheduler
      securityContext: {}
      serviceAccount: kubernetes-dashboard
      serviceAccountName: kubernetes-dashboard
      terminationGracePeriodSeconds: 30
      tolerations:
      - effect: NoSchedule
        key: node-role.kubernetes.io/master
      volumes:
      - name: kubernetes-dashboard-certs
        secret:
          defaultMode: 420
          secretName: kubernetes-dashboard-certs
      - emptyDir: {}
        name: tmp-volume
status:
  availableReplicas: 1
  conditions:
  - lastTransitionTime: "2020-01-06T06:27:23Z"
    lastUpdateTime: "2020-01-06T06:27:23Z"
    message: Deployment has minimum availability.
    reason: MinimumReplicasAvailable
    status: "True"
    type: Available
  - lastTransitionTime: "2020-01-06T06:21:35Z"
    lastUpdateTime: "2020-01-06T06:27:23Z"
    message: ReplicaSet "kubernetes-dashboard-7c54d59f66" has successfully progressed.
    reason: NewReplicaSetAvailable
    status: "True"
    type: Progressing
  observedGeneration: 1
  readyReplicas: 1
  replicas: 1
  updatedReplicas: 1
```

### type字段
type相关的字段的定义可以参考 [kubernetes apimachinery meta/v1/types TypeMeta](https://github.com/kubernetes/apimachinery/blob/master/pkg/apis/meta/v1/types.go#L41) 主要是以下字段

1. apiVersion: 使用的API对象的版本，可以参考kubernetes github源码 [v1beta1](https://github.com/kubernetes/api/tree/kubernetes-1.17.0/extensions/v1beta1) 就是表示版本
2. kind: API对象类型，对于Deployment来说一直是 Deployment

### meta字段
meta相关的字段的定义可以参考 [kubernetes apimachinery meta/v1/types ObjectMeta](https://github.com/kubernetes/apimachinery/blob/master/pkg/apis/meta/v1/types.go#L110) 主要是以下字段

1. name: Deployment名称，一般由业务在yaml文件里面设置
2. namespace: Deployment所属的namespace （kubernetes namespace类似组的概念，和linux namespace不同）
3. creationTimestamp: Pod创建时间
4. labels: Pod相关的label，有些是用户设置的，有些则是kubernetes自动设置的
5. uid: kubernetes自动生成的唯一的pod id
6. selfLink: 当前pod的url，可以通过访问该url获取到pod的相关信息
7. annotations: 用户自己设置的一些key-value键值对注释，类似labels

### spec字段
spec相关的字段的定义可以参考 [kubernetes api extensions/v1beta1/types DeploymentSpec](https://github.com/kubernetes/api/blob/kubernetes-1.17.0/extensions/v1beta1/types.go#L101) 主要是以下字段

1. replicas: Deployment锁期望的Pod数
2. selector: Deployment控制器用于筛选Pod的label，一般和template内metadata.labels保持一致
3. revisionHistoryLimit: 要保留多少个ReplicaSet的版本，默认为10
4. strategy: Deployment的升级策略，主要有以下2种 `Recreate` 和 `RollingUpdate`
5. template: Pod template，用于描述Pod，具体可以参考 [kubernetes pod属性](https://chenguolin.github.io/2019/03/27/Kubernetes-17-Kubernetes%E4%B9%8BPod/#-pod%E5%B1%9E%E6%80%A7)
6. paused: 标识Deployment是否被暂停

### status字段
status相关的字段的定义可以参考 [kubernetes api extensions/v1beta1/types DeploymentStatus](https://github.com/kubernetes/api/blob/kubernetes-1.17.0/extensions/v1beta1/types.go#L238) 主要是以下字段

1. replicas: 当前Deployment控制器selector match的Pod个数，通常和spec.replicas数值一样
2. availableReplicas: 当前可用的Pod个数，通常和replicas数值一样
3. updatedReplicas: 更新到最新状态的Pod个数
4. readyReplicas: ready状态的Pod个数
5. conditions: Deployment的状态列表，可以参考 [kubernetes api extensions/v1beta1/types DeploymentCondition](https://github.com/kubernetes/api/blob/kubernetes-1.17.0/extensions/v1beta1/types.go#L295)

## ③ 滚动升级
从上面的 spec 字段里面我们知道 spec.strategy 字段可以用来控制Deployment的升级策略，在实际生产过程中我们用的最多的是 `RollingUpdate` 也就是 滚动升级，对于Deployment来说只要 Pod template 有变更就会触发 Deployment 升级。（必须要求Deployment spec.strategy 是RollingUpdate类型）

变更 Deployment 的 Pod template 有以下几种方式

1. 通过 `kubectl set` 命令进行变更，例如 `kubectl set image deployment/nginx-deployment nginx=nginx:1.9.1 --record` 变更镜像版本
2. 通过 `kubectl edit` 命令直接编辑，例如 `kubectl edit deployment.v1.apps/nginx-deployment` 变更 Pod template

变更完成之后，我们可以通过 `kubectl rollout status deployment.v1.apps/nginx-deployment` 查看当前升级的进度

## ④ 回滚
很多时候我们经常会遇到升级后发现新版本有问题，需要进行快速回滚到上一个稳定版本，kubernetes提供了非常遍历的命令方便用户进行 Deployment 回滚。每次Deployment升级成功之后会生成一个新的版本。（只有 Pod template 变更升级才会生成新的版本）

1. 查看Deployment的版本历史  `kubectl rollout history -n kube-system deployment {deployment-name}`
2. 查看历史版本信息 `kubectl rollout history -n kube-system deployment {deployment-name} --revision=2`
3. 回滚到指定版本  `kubectl rollout undo deployment.v1.apps/nginx-deployment --to-revision=2

## ⑤ 弹性伸缩
除了上诉提到的功能之外，Deployment还有一个非常重要的功能就是 弹性伸缩。很多人选择使用kubernetes容器集群目的就是希望能够利用容器平台的弹性伸缩功能，除了节点粒度的弹性伸缩，还可以在Pod维度做弹性伸缩，Deployment的弹性伸缩正是Pod维度的。

我们知道流量是有高峰低峰之分的，传统物理机时代为了应对流量高峰我们通常是买了一堆机器，当流量下去之后这些机器就空闲着，非常的浪费。而有了kubernetes容器平台之后，通过Node、Pod维度的弹性伸缩，能够做到资源的最大化利用，同时保证成本最低。

Deployment 弹性伸缩相关的命令 为 `kubectl scale deployment -n kube-system {deployment-name} --replicas=2`

初此之外，生产环境我们会开启 [HPA](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale-walkthrough/)，根据所有的Pod的CPU利用率，然后计算是要缩容还是扩容，例如这个命令可以开启HPA `kubectl autoscale deployment -n kube-system {deployment-name} --cpu-percent=50 --min=1 --max=10`。

