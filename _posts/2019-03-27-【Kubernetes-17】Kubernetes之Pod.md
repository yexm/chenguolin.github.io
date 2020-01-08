---
layout:  post  # 使用的布局（不需要改）
catalog: true  # 是否归档
author: 陈国林 # 作者
tags:          #标签
    - Kubernetes
---

# 一. 背景
前面关于Docker的文章里面我们提到了 `Docker容器本质是进程，它主要由Namespace、Cgroups、Rootfs这三种技术实现。Docker是最有名的容器运行时，它可以在宿主机上启动很多容器`。在 [kubernetes-对象简单介绍](https://chenguolin.github.io/2019/03/25/Kubernetes-14-Kubernetes%E5%AF%B9%E8%B1%A1%E7%AE%80%E5%8D%95%E4%BB%8B%E7%BB%8D/) 文章里面提到 `Pod是Kubernetes最小的调度单位`，可能大家都会很好奇 Pod 到底是什么，它和 Docker 的关系是怎么样的。所以，这篇文章会介绍 Kubernetes Pod，了解 Pod 是什么。

让我们来考虑一个使用场景 `假设有2个进程 P1和P2，为了协同工作，要求P1和P2必须部署在同一台机器，同时要能够网络互通，以及共享文件存储`，针对这种场景我们看下如果使用 Docker 部署的话该怎么实现。

显然我们需要运行2个容器，每个容器内跑进程P1和P2，两个容器需要使用相同的网络配置，同时需要挂载相同的Volume。另外我们还需要考虑2个容器的启动顺序，因为可能进程间会有启动依赖。

```这里不推荐一个容器内同时运行P1和P2，因为容器是单进程模型，也就是说容器没有管理多个进程的能力。这是因为容器里 PID=1 的进程就是应用本身，其他的进程都是这个 PID=1 进程的子进程。如果PID=1进程不会给子进程传系统信号的话，子进程是没有办法正常接收到系统信号的，也就是没有办法优雅退出，存在风险，详细可以参考 https://chenguolin.github.io/2019/06/22/Kubernetes-30-Kubernetes容器应用优雅退出机制/```

Docker的实现方案虽然可行，但是很麻烦，因为我们不仅要考虑多个容器共享网络、共享存储，还要考虑容器的启动顺序。但是到了 Kubernetes ，这样的问题就迎刃而解了，`Pod 是 Kubernetes 里的原子调度单位。这就意味着，Kubernetes 项目的调度器，是统一按照 Pod 而非容器的资源需求进行计算的`。

所以，在 Kubernetes 里面实现方案为 把2个容器绑定在同一个Pod上，同一个Pod 里的所有容器，共享同一个 Network Namespace，并且可以声明共享同一个 Volume。

Docker是kubernetes项目使用最多的容器运行时，但是kubernetes还支持其他的容器运行时 [container-runtimes](https://kubernetes.io/docs/setup/production-environment/container-runtimes/)，在kubernetes中使用Pod主要是以下2种方式。

1. `Pod内运行单容器`: 最常见的使用方式，相当于Pod封装了一下容器，kubernetes直接管理Pod而非容器
2. `Pod内运行多个容器`: 可以把多个关联的容器放在同一个Pod内，同一个Pod 里的所有容器，共享同一个 Network Namespace，可以使用localhost进行通信

# 二. Pod
Kubernetes 中 Pod 的实现需要使用一个中间容器，这个容器叫作 Infra 容器。在这个 Pod 中，Infra 容器永远都是第一个被创建的容器，而其他用户定义的容器，则通过 Join Network Namespace 的方式，与 Infra 容器关联在一起。

在 Kubernetes 项目里，Infra 容器一定要占用极少的资源，所以它使用的是一个非常特殊的镜像，叫作 `k8s.gcr.io/pause`。这个镜像是一个用汇编语言编写的、永远处于 暂停 状态的容器，解压后的大小也只有 100~200 KB 左右。当 Infra 容器设置了 Network Namespace 后，用户容器就可以加入到 Infra 容器的 Network Namespace 当中了。

我们可以来验证一下

1. 查看 apiserver pod
   ```
   $ kubectl get pods -n kube-system -o wide | grep kube-apiserver-ecs-s6-large-2-linux-20200105130533
   kube-apiserver-ecs-s6-large-2-linux-20200105130533            1/1     Running   2          33h   192.168.0.14    ecs-s6-large-2-linux-20200105130533   <none>           <none>
   ```

2. 登录对应机器查看 docker容器进程  （发现有2个进程，其中5a894a10c697进程确实用pause镜像启动，为什么不是用k8s.gcr.io/pause。因为国内无法下载，用阿里云的镜像替代）
   ```
   $ docker ps -a | grep apiserver
   7589296a202d        0cae8d5cc64c                                        "kube-apiserver --ad…"   29 hours ago        Up 29 hours                                     k8s_kube-apiserver_kube-apiserver-ecs-s6-large-2-linux-20200105130533_kube-system_212183bed1727623dc1edaf897053c50_2
   5a894a10c697        registry.aliyuncs.com/google_containers/pause:3.1   "/pause"                 29 hours ago        Up 29 hours                                     k8s_POD_kube-apiserver-ecs-s6-large-2-linux-20200105130533_kube-system_212183bed1727623dc1edaf897053c50_2
   ```
   
3. 查看两个进程对应的 network namespace
   ```
   $ docker inspect 7589296a202d | grep Pid
     "Pid": 2861,
   $ ls -lsrt /proc/2861/ns/
   total 0
   0 lrwxrwxrwx 1 root root 0 1月   7 19:20 uts -> uts:[4026531838]
   0 lrwxrwxrwx 1 root root 0 1月   7 19:20 user -> user:[4026531837]
   0 lrwxrwxrwx 1 root root 0 1月   7 19:20 pid -> pid:[4026532263]
   0 lrwxrwxrwx 1 root root 0 1月   7 19:20 net -> net:[4026531956]
   0 lrwxrwxrwx 1 root root 0 1月   7 19:20 mnt -> mnt:[4026532262]
   0 lrwxrwxrwx 1 root root 0 1月   7 19:20 ipc -> ipc:[4026532246]
   
   $ docker inspect 5a894a10c697  | grep Pid
     "Pid": 2651
   $ ls -lsrt /proc/2651/ns/
   total 0
   0 lrwxrwxrwx 1 root root 0 1月   7 19:21 uts -> uts:[4026532245]
   0 lrwxrwxrwx 1 root root 0 1月   7 19:21 user -> user:[4026531837]
   0 lrwxrwxrwx 1 root root 0 1月   7 19:21 pid -> pid:[4026532247]
   0 lrwxrwxrwx 1 root root 0 1月   6 13:47 net -> net:[4026531956]
   0 lrwxrwxrwx 1 root root 0 1月   7 19:21 mnt -> mnt:[4026532244]
   0 lrwxrwxrwx 1 root root 0 1月   6 13:47 ipc -> ipc:[4026532246]
   ```
   
4. 通过第3步，我们可以确认容器进程 7589296a202d 和 5a894a10c697 对应的 user、net、ipc namespace相同，因此证实了 Infra容器 和 用户容器确实是共享network namespace。

`这也就意味着，如果 Pod 里的有多个容器，那么这些容器可以直接使用 localhost 进行通信，同时它们看到的网络设备跟 Infra 容器看到的完全一样。`

除了网络之外，同一个Pod内的容器如果要共享存储，可以通过Volume来实现，容器挂载相关的Volume即可实现共享存储，如下面这个例子。nginx-container 和 debian-container 容器都挂载了 shared-data volume，而 shared-data 对应的是 hostPath volume，实际上是宿主机的 /data 目录。所以，通过这种方式两个容器就可以共享宿主机 /data 达到共享存储的目的。

```
apiVersion: v1
kind: Pod
metadata:
  name: two-containers
spec:
  restartPolicy: Never
  volumes:
  - name: shared-data
    hostPath:
      path: /data
  containers:
  - name: nginx-container
    image: nginx
    volumeMounts:
    - name: shared-data
      mountPath: /usr/share/nginx/html
  - name: debian-container
    image: debian
    volumeMounts:
    - name: shared-data
      mountPath: /pod-data
    command: ["/bin/sh"]
    args: ["-c", "echo Hello from the debian container > /pod-data/index.html"]
```

# 三. 使用
## ① 常用命令
1. 运行Pod `kubectl apply -f pod.yaml`
2. 查看Pod列表 `kubectl get pod -n {namespace}`
3. 删除Pod `kubectl delete po -n {namespace} {pod-name}`
4. 查看Pod yaml `kubectl get pod -n {namespace} {pod-name} -o yaml`
5. 描述Pod `kubectl describe po -n {namespace} {pod-name}`
6. 查看Pod 日志 `kubectl logs -f --tail 1000 -n {namespace} {pod-name}`  如果有多个容器可以用这个命令查看某个容器日志 `kubectl logs -f -n kube-system {pod-name} -c {container-name}`
7. 进入Pod `kubectl exec -it -n {namespace} {pod-name} /bin/sh`

## ② Pod属性
我们知道任何kubernetes对象都可以通过 yaml 文件进行描述，Pod也不例外。为了更好的了解Pod，我们需要熟悉Pod yaml 配置文件相关属性字段的含义，我们通过
下面这个例子来了解一下Pod。有关 Pod 属性字段的相关含义，可以参考 [kubernetes api core/v1/types](https://github.com/kubernetes/api/blob/master/core/v1/types.go#L3513)

```
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: "2020-01-06T06:21:35Z"
  generateName: kubernetes-dashboard-7c54d59f66-
  labels:
    k8s-app: kubernetes-dashboard
    pod-template-hash: 7c54d59f66
  name: kubernetes-dashboard-7c54d59f66-l5f6d
  namespace: kube-system
  ownerReferences:
  - apiVersion: apps/v1
    blockOwnerDeletion: true
    controller: true
    kind: ReplicaSet
    name: kubernetes-dashboard-7c54d59f66
    uid: e4675c10-f794-4900-afed-61700a34676c
  resourceVersion: "37589"
  selfLink: /api/v1/namespaces/kube-system/pods/kubernetes-dashboard-7c54d59f66-l5f6d
  uid: f4eefe2c-c0c2-49fb-9015-c1a79b006b03
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
    - mountPath: /var/run/secrets/kubernetes.io/serviceaccount
      name: kubernetes-dashboard-token-6scf4
      readOnly: true
  dnsPolicy: ClusterFirst
  enableServiceLinks: true
  nodeName: ecs-s6-large-2-linux-20200105130533
  priority: 0
  restartPolicy: Always
  schedulerName: default-scheduler
  securityContext: {}
  serviceAccount: kubernetes-dashboard
  serviceAccountName: kubernetes-dashboard
  terminationGracePeriodSeconds: 30
  tolerations:
  - effect: NoSchedule
    key: node-role.kubernetes.io/master
  - effect: NoExecute
    key: node.kubernetes.io/not-ready
    operator: Exists
    tolerationSeconds: 300
  - effect: NoExecute
    key: node.kubernetes.io/unreachable
    operator: Exists
    tolerationSeconds: 300
  volumes:
  - name: kubernetes-dashboard-certs
    secret:
      defaultMode: 420
      secretName: kubernetes-dashboard-certs
  - emptyDir: {}
    name: tmp-volume
  - name: kubernetes-dashboard-token-6scf4
    secret:
      defaultMode: 420
      secretName: kubernetes-dashboard-token-6scf4
status:
  conditions:
  - lastProbeTime: null
    lastTransitionTime: "2020-01-06T06:21:35Z"
    status: "True"
    type: Initialized
  - lastProbeTime: null
    lastTransitionTime: "2020-01-06T06:27:23Z"
    status: "True"
    type: Ready
  - lastProbeTime: null
    lastTransitionTime: "2020-01-06T06:27:23Z"
    status: "True"
    type: ContainersReady
  - lastProbeTime: null
    lastTransitionTime: "2020-01-06T06:21:35Z"
    status: "True"
    type: PodScheduled
  containerStatuses:
  - containerID: docker://0c3688c4115547af792019d21a4e7f1ff9ff1ddf5555a81d41d4d29cdcc437f4
    image: k8s.gcr.io/kubernetes-dashboard-amd64:v1.10.1
    imageID: docker-pullable://registry.cn-hangzhou.aliyuncs.com/google_containers/kubernetes-dashboard-amd64@sha256:0ae6b69432e78069c5ce2bcde0fe409c5c4d6f0f4d9cd50a17974fea38898747
    lastState: {}
    name: kubernetes-dashboard
    ready: true
    restartCount: 0
    started: true
    state:
      running:
        startedAt: "2020-01-06T06:27:22Z"
  hostIP: 192.168.0.14
  phase: Running
  podIP: 10.244.0.12
  podIPs:
  - ip: 10.244.0.12
  qosClass: BestEffort
  startTime: "2020-01-06T06:21:35Z"
```

### type字段
type相关的字段的定义可以参考 [kubernetes apimachinery meta/v1/types TypeMeta](https://github.com/kubernetes/apimachinery/blob/master/pkg/apis/meta/v1/types.go#L41) 主要是以下字段

1. apiVersion: 使用的API对象的版本，可以参考kubernetes github源码 [v1](https://github.com/kubernetes/api/tree/master/core/v1) 就是表示版本
2. kind: API对象类型，对于Pod来说一直是 Pod

### meta字段
type相关的字段的定义可以参考 [kubernetes apimachinery meta/v1/types ObjectMeta](https://github.com/kubernetes/apimachinery/blob/master/pkg/apis/meta/v1/types.go#L110) 主要是以下字段

1. name: Pod名称，kubernetes能够保证每个pod名称独一无二
2. namespace: Pod所属的namespace （kubernetes namespace类似组的概念，和linux namespace不同）
3. creationTimestamp: Pod创建时间
4. labels: Pod相关的label，有些是用户设置的，有些则是kubernetes自动设置的
5. ownerReferences: 有些Pod是由更顶层的API对象例如ReplicaSet等控制创建的，这个字段标识它的上一层API对象是谁
6. uid: kubernetes自动生成的唯一的pod id
7. selfLink: 当前pod的url，可以通过访问该url获取到pod的相关信息
8. annotations: 用户自己设置的一些key-value键值对注释，类似labels

### spec字段
spec相关的字段定义可以参考 [kubernetes api core/v1/types PodSpec](https://github.com/kubernetes/api/blob/master/core/v1/types.go#L2833) 主要是以下字段

1. initContainers: init容器相关的字段，kubernetes提供了init container机制，只有所有的init container都成功执行后才会开始执行业务容器
2. containers: 业务容器相关的字段 
    + name: 容器名称
    + image: 镜像地址
    + imagePullPolicy: 镜像拉取策略，目前支持 `Always`、`Never`、`IfNotPresent` 这三种策略
    + command: 容器运行命令
    + args: 容器运行命令的参数列表
    + ports: 容器暴露的端口
    + env: 容器内设置的环境变量
    + resources: 容器资源申请，详情可以参考 [manage-compute-resources-container](https://kubernetes.io/docs/concepts/configuration/manage-compute-resources-container/)
    + volumeMounts: 容器Volume挂载
    + livenessProbe: 容器存活探针
    + readinessProbe: 容器就绪探针
    + lifecycle: 配置容器生命周期，主要有 postStart 和 preStop 两个hook
3. nodeName: 当前Pod调度到的节点，当成功调度到某个节点之后会被kubernetes自动设置的
4. restartPolicy: Pod重启策略，目前支持 `Always`、`OnFailure`、`Never` 这3种策略
5. securityContext: Pod安全相关设置，默认情况下未空
6. serviceAccountName: 当前Pod访问apiserver用的service account
7. terminationGracePeriodSeconds: Pod优雅退出超时时间，默认为30s，关于容器的优雅退出可以参考 [Kubernetes容器应用优雅退出机制](https://chenguolin.github.io/2019/06/22/Kubernetes-30-Kubernetes%E5%AE%B9%E5%99%A8%E5%BA%94%E7%94%A8%E4%BC%98%E9%9B%85%E9%80%80%E5%87%BA%E6%9C%BA%E5%88%B6/#%E4%BA%8C-%E5%AE%B9%E5%99%A8%E4%BC%98%E9%9B%85%E9%80%80%E5%87%BA)
8. tolerations: Pod容忍了哪些污点
9. volumes: kubernetes volume配置

### status字段
status相关的字段定义可以参考 [kubernetes api core/v1/types PodStatus](https://github.com/kubernetes/api/blob/master/core/v1/types.go#L3401) 主要是以下字段

1. phase: Pod的阶段，主要有以下5个
   ```
   1). Pending     // Pod已经被kubernetes成功调度，但是有一个或多个容器还未创建，常见的原因有 镜像下载、资源不足
   2). Running     // Pod正在运行，所有的容器都已经完成创建，至少有一个容器还在运行中
   3). Succeeded   // Pod执行成功，所有的容器都已经成功推出
   4). Failed      // Pod执行失败，所有容器都已经结束，至少有一个容器运行失败 （容器退出返回码非0）
   5). Unknown     // Pod状态未知，当Pod宿主机网络不通的时候会出现这种情况 （kubelet 和 apiserver无法通信） 
   ```
2. podIP: 当前Pod的IP，每个Pod都有一个IP，一般Pod的IP段是 10.244.0.0/16
3. hostIP: 宿主机IP
4. startTime: Pod启动时间
5. conditions: Pod的状态，最重要的是 [containerStatuses](https://github.com/kubernetes/api/blob/master/core/v1/types.go#L2378) 这个字段
    + containerStatuses.containerID: 容器ID
    + containerStatuses.image: 容器镜像
    + containerStatuses.name: 容器名称
    + containerStatuses.ready: 容器是否ready
    + containerStatuses.restartCount: 容器重启次数 （注意容器重启 Pod并不会重启，两者生命周期不同）
    + containerStatuses.state: 容器状态

# 四. 流程
## ① 创建Pod

## ② 删除Pod
由于Pod封装了容器，而容器本质是在宿主机上运行的进程，因此删除Pod的时候需要考虑如何保证容器进程优雅退出，kubernetes删除Pod的流程大致如下所示

1. 用户发送 delete pod的请求（可以通过kubectl delete 命令实现）
2. 默认情况下 pod 的 terminationGracePeriodSeconds 时间为30s，用户可以通过修改 pod yaml 调整这个优雅退出的超时时间
3. 当 apiserver 收到删除pod的请求后，会更新 etcd 的meta信息
4. kubelet 发现 pod 被标记为 terminating 后，如果容器配置了 preStop hook，那么会先执行 preStop，直到这个 Hook 定义操作完成之后，才允许容器被杀死，这跟 postStart 不一样。[kubelet kuberuntime killContainer](https://github.com/kubernetes/kubernetes/blob/master/pkg/kubelet/kuberuntime/kuberuntime_container.go#L552)
5. terminationGracePeriodSeconds 减去 preStop 执行时间为容器进程优雅退出时间，如果 preStop 执行超过terminationGracePeriodSeconds 则容器进行优雅退出时间会被强制设置为 2秒 [kubelet kuberuntime set gracePeriod](https://github.com/kubernetes/kubernetes/blob/master/pkg/kubelet/kuberuntime/kuberuntime_container.go#L592)
6. kubelet 通过CRI 接口发送 [StopContainer](https://github.com/kubernetes/kubernetes/blob/master/pkg/kubelet/kuberuntime/kuberuntime_container.go#L602) rpc请求，并带上超时时间配置 
7. 容器运行时先发送 SIGTERM 信号，如果过了超时时间容器进程还未介绍，则发送 SIGKILL 强制 kill 掉容器进程 [containerd cri container stop](https://github.com/containerd/cri/blob/95bd02d28f28c95154755e9de095615717afc14f/pkg/server/container_stop.go#L103)

`建议: 如果容器设置了 preStop hook，建议把terminationGracePeriodSeconds配置大一点，如果 preStop 运行太久可能会导致容器进程被操作系统强杀，导致没有办法优雅退出，影响业务。`

