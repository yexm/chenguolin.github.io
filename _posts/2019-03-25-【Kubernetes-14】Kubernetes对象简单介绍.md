---
layout:  post  # 使用的布局（不需要改）
catalog: true  # 是否归档
author: 陈国林 # 作者
tags:          #标签
    - Kubernetes
---

# 一. Pod
1. 什么是Pod?
    * Pod是K8s中创建或部署的最小单位，每个Pod将被绑定调度到Node节点上，并一直保持在那里直到被终止或删除。
    * Pod是一种相对短暂的存在，而不是持久存在的。Pod被调度到Node节点上，当一个Node节点销毁或挂掉之后，上面的所有Pod都会被删除。
    * Pod是由一个或多个相关的容器组成，同一个Pod中的容器可以共享网络、文件系统等。
    * K8s集群中的每个Pod都有一个独立的IP地址，可保证Pod可以和其它物理机及其他容器进行无障碍通信。
    * Pod本身不具备自愈能力，一般情况下使用Deployment、ReplicaSet、Replication Controller等来创建和管理Pod，保证有足够的Pod副本在运行。
    * 重启Pod中的容器和重启Pod不是一回事，Pod只提供容器的运行环境并保持容器的运行状态，重启容器不会造成Pod重启。
2. K8s使用Pod2种方式
    * Pod中运行一个容器，这是K8s中最常见的用法。可以将Pod视为单个封装的容器，只是K8s是直接管理Pod而不是容器。
    * Pod中运行多个容器，多个容器共享Pod资源。这些容器可以形成单一的内部Service。 
3. Pod共享2种资源
    * `网络`: 每个Pod被分配一个独立的IP地址，Pod中的每个容器共享网络命名空间，包括IP地址和网络端口。Pod内的容器可以使用localhost相互通信。当Pod中的容器与Pod外部通信时，他们必须协调如何使用共享网络资源。
    * `存储`: Pod可以指定一组共享存储Volumes，Pod中的所有容器都可以访问共享Volumes允许这些容器共享数据。Volumes还用于Pod中的数据持久化，以防其中一个容器需要重新启动而丢失数据。
4. Pod状态
    * `Pending`: 挂起, Pod已经被K8s系统接受，但有一个或者多个容器镜像尚未创建，等待时间包括下载镜像时间和Pod调度时间。
    * `Running`: 运行中，该Pod已经绑定到一个Node上，Pod中所有容器都已被创建，至少有一个容器正在运行，或者正处于启动或重启状态。
    * `Succeed`: 成功，Pod中所有容器都被成功结束，并且不会再重启。
    * `Failed`: 失败，Pod中所有容器都终止了，并且至少有一个容器因为失败而终止。
    * `Unknown`: 未知，因为某些原因无法取得Pod的状态，通常是因为Pod与所在Node通信失败。
5. Pod创建  
   因为Pod不会自愈，如果Node节点有故障这个Pod就会被删除，或者Node节点缺少资源情况下Pod也会被删除。所以建议采用相关Controller来创建Pod而不是直接创建Pod，因为单独的Pod没有办法自愈，而Controller却可以。
   
   有3种可用的Controller
     * `Job`: 使用Job运行预期会结束的Pod，例如批量计算，Job仅适用于重启策略为OnFailure或Never的Pod。
     * `Deployment、ReplicaSet、Replication Controller`: 预期不会终止的Pod使用这3个Controller来创建，Replication Controller仅适用于具有restartPolicy为Always的Pod。
     * `DaemonSet`: 提供特定于机器的系统服务，DaemonSet为每台机器运行一个Pod
6. Pod容器重启  
   Pod YAML配置文件Spec中有一个`restartPolicy`字段，有3个值为 Always、OnFailure 和 Never，默认为Always。restartPolicy适用于Pod中的所有容器，restartPolicy仅指通过同一节点上的kubelet重新启动容器。失败的容器由kubelet以五分钟为上限的指数退避延迟（10秒，20秒，40秒…）重新启动，并在成功执行十分钟后重置。

# 二. Node
![](https://github.com/chenguolin/chenguolin.github.io/blob/master/data/image/k8s-node.png?raw=true)

1. 什么是Node  
   `Node`是K8s中的工作节点，可以是虚拟机或物理机。每个Node由K8s Master管理，Node上可以有多个Pod，K8s Master会自动处理Node的Pod调度，同时Master的自动调度会考虑每个Node上的可用资源，为了管理Pod，每个Node上至少要运行Docker、kubelet和kube-proxy。
2. Node管理
    * Node本质不是由K8s来创建的，K8s只是管理Node上的资源，默认情况下kubelet在启动的时候会向master注册自己，并创建Node资源。
    * Node Controller负责以下几个点
        * 维护Node状态
        * 与Cloud Provider同步Node
        * 给Node分配容器CIDR
        * 删除带有NoExecute taint的Node上的Pods
3. Node状态信息
     * 基本信息: 包括内核版本、容器引擎版本、OS类型
     * 地址: 包括hostname、内网IP和外网IP
     * 容量: Node上的可用资源，CPU、内存和Pod总数
     * 条件: 包括OutOfDisk、Ready、MemoryPressure和DiskPressure
     
# 三. Service
![](https://github.com/chenguolin/chenguolin.github.io/blob/master/data/image/k8s-service.svg)

1. 什么是Service
    * K8s中的Service是一个抽象的概念，它定义了Pod的逻辑分组和一种可以访问它们的策略，这组Pod能被Service访问，使用YAML（优先）或JSON 来定义Service，Service所针对的一组Pod通常由LabelSelector实现。 
    * Service是一个抽象层，它定义了一组逻辑的Pods，这些Pod提供了相同的功能，借助Service，应用可以方便的实现服务发现与负载均衡，可以把Service加上一组Pod称作是一个微服务。
    * 默认情况Pod只能通过K8s集群内部IP访问，要使Pod允许从K8s虚拟网络外部访问，须要使用K8s Service暴露Pod。
    * 当创建一个Service的时候每个Service被分配一个唯一的IP地址，这个IP地址与一个Service的生命周期绑定在一起，当Service存在的时候它不会改变。
    * 每个节点都运行了一个kube-proxy，kube-proxy监控着K8s增加和删除Service，对于每个Service kube-proxy会随机开启一个本机端口，任何向这个端口的请求都会被转发到一个后台的Pod中，如何选择哪一个后台Pod是基于SessionAffnity进行的分配。
    * 在创建Service的时候可以指定IP地址，将spec.clusterIP的值设置为我们想要的IP地址即可。
2. 创建一个Service
    * 先创建一个Deployment
      ```
      apiVersion: extensions/v1beta1
      kind: Deployment
      metadata:
        name: hostnames
      spec:
        selector:
          app: hostnames
        replicas: 3
        template:
          metadata:
            labels:
              app: hostnames
          spec:
            containers:
            - name: hostnames
              image: k8s.gcr.io/serve_hostname
              ports:
              - containerPort: 9376
                protocol: TCP
      ```
      `$ kubectl apply -f deployment.yaml`
      
      这个效果和以下启动命令效果是一样的  
      `$ kubectl run hostnames --image=k8s.gcr.io/serve_hostname \`  
                        `--labels=app=hostnames \`  
                        `--port=9376 \`  
                        `--replicas=3``
     * 确认Pod是否都是Running状态  
       `$ kubectl get pods -l app=hostnames`
     * 创建一个Service  
       `$ kubectl expose deployment myservice --port=80 --target-port=9376`
     * 确认Service是否存在  
       `$ kubectl get svc myservice`
     * 测试DNS是否能够解析myservice  
       `$ kubectl run curl --image=radial/busyboxplus:curl -i —tty`  
       `$ nslookup myservice`  
         Server:    10.96.0.10  
         Address 1: 10.96.0.10 kube-dns.kube-system.svc.cluster.local  
         
         Name:      myservice  
         Address 1: 10.101.147.211 myservice.default.svc.cluster.local  
3. Service 服务发现  
   K8s支持2种方式来发现服务，环境变量和DNS
    * 当一个Pod在一个Node上运行的时候，Kubelet会针对运行的Service增加一序列的环境变量，它支持Docker links compatible和普通环境变量，比如redis-master Service暴露了TCP 6379端口，并且被分配了10.0.0.11 IP地址，那么会有如下环境变量
        * REDIS_MASTER_SERVICE_HOST=10.0.0.11
        * REDIS_MASTER_SERVICE_PORT=6379
        * REDIS_MASTER_PORT=tcp://10.0.0.11:6379
        * REDIS_MASTER_PORT_6379_TCP=tcp://10.0.0.11:6379
        * REDIS_MASTER_PORT_6379_TCP_PROTO=tcp
        * REDIS_MASTER_PORT_6379_TCP_PORT=6379
        * REDIS_MASTER_PORT_6379_TCP_ADDR=10.0.0.11  
      `所有想要被Pod访问的Service都需要在Pod创建之前创建，否则这个环境变量是没有的，使用DNS则没有这个问题`
    * DNS是云平台插件，DNS服务器监控着API Server，当有Service被创建的时候，DNS服务器会为之创建相应的记录，比如我们在my-ns的namespace下创建一个Service为my-service，这个时候DNS就会自动创建一个my-service.my-ns的记录，所有my-ns下的Pod都可以通过域名my-service查询找到对应的IP地址，不同namespace下的Pod查找必须使用my-service.my-ns才可以。

# 四. Label
1. 什么是Label
    * label是一对key/value被关联到对象上比如Pod，标签的使用我们倾向于能够标识对象的特点，并且对用户而言是有意义的，但是标签对内核系统是没有直接意义的。
    * 标签可以用来划分特定组的对象，标签可以在创建一个对象的时候直接给与也可以在后期随时修改，每一个对象可以拥有多个标签，但是key值必须是唯一的。
2. Label语法和字符集
    * label是一对key/value，有效的key=一个可选的前缀 + 名称组成，通过/来区分
    * 名称部分是必须的，最多63个字符，开始和结束的字符必须是字母或者数字，中间是字母或数字以及特殊的字符`_,`-,`.`，前缀是可有可无
    * 如果指定了前缀，那前缀必须是一个DNS子域，一序列的DNS label通过`.`来划分，长度不超过253个字符，`/`结尾
    * label的值必须小于或等于63个字符，允许为空。如果值不为空，那么首位字符必须为字母数字，中间必须是数字字母以及特殊字符`-`,`_`,`.`
3. Label选择器
    * 基于运算符，有3种运算符是允许的`=`,`==`和`!=`，`=`和`==`是同一种意思
        * environment = production  （选择environment等于production的资源)
        * tier != frontend    (选择tier不等于frontend的资源)
    * 基于set的条件，支持3种操作符 in, notin 和 exists(仅针对key)
        * environment in (production, qa)   (选择environment等于production或者qa的资源)
        * tier notin (frontend, backend)  (选择tier不等于frontend和backend的资源)
        * partition     (选择key为partition的资源，不care value是什么)
        * !partition  (选择key不是partition的资源，不care value是什么)
4. kubectl 使用Label命令示例
    * kubectl get pods -l environment=production,tier=frontend
    * kubectl get pods -l 'environment in (production),tier in (frontend)’

# 五. Volume
1. K8s Volume解决什么问题
    * 容器中磁盘的生命周期是短暂的当一个容器损坏之后Kubelet会重启这个容器，会导致文件丢失
    * 当很多容器在同一个Pod中运行，很多时候需要共享文件
2. Volume特点
    * 一个Volume拥有明确的生命周期，与所在的Pod的生命周期相同，因此Volume的生命周期比Pod中运行的任何容器都要持久
    * Volume与Pod相关和容器无关，因此容器重启的时候数据还会保留，如果Pod被删除那么这些数据也会被删除
    * 内部实现中一个Volume就是一个目录，它可能包含一些数据，这些数据对Pod中的容器都是可用的
    * 想要使用一个Volume，Pod必须指明Pod提供了哪些磁盘，并且说明如何挂载到容器中
3. K8s支持的Volume类型
    * emptyDir: 使用emptyDir，当Pod分配到Node上时将会创建emptyDir，并且只要Node上的Pod一直运行，Volume就会一直存在。当Pod从Node上被删除时，emptyDir也同时会删除，存储的数据也将永久删除，删除容器不影响emptyDir。
    * hostPath: hostPath允许挂载Node上的文件系统到Pod里面去。如果Pod需要使用Node上的文件，可以使用hostPath。
    * gcePersistentDisk
    * awsElasticBlockStore
    * nfs
    * iscsi
    * fc (fibre channel)
    * flocker
    * glusterfs
    * rbd
    * cephfs
    * gitRepo
    * secret
    * persistentVolumeClaim
    * downwardAPI
    * projected
    * azureFileVolume
    * azureDisk
    * vsphereVolume
    * Quobyte
    * PortworxVolume
    * ScaleIO
    * StorageOS
    * local

# 六. Deployment
1. 什么是Deployment  
   Deployment为Pod和ReplicaSet提供了一个声明式定义方法，用来替代以前的Replication Controller以便管理应用，典型的应用如下
    * 定义Deployment来创建Pod和ReplicaSet
    * 滚动升级和回滚应用
    * 扩容和缩容
    * 暂停和继续Deployment
2. 创建一个Deployment
   ```
   apiVersion: extensions/v1beta1
   kind: Deployment
   metadata:
    name: nginx-deployment
   spec:
    replicas: 3
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
    `$ kubectl apply -f nginx.yaml`
3. Deployment状态
    * Progressing: 进行中
        * Deployment正在创建新的ReplicaSet
        * Deployment正在扩容一个已有的ReplicaSet
        * Deployment正在缩容一个已有的ReplicaSet
    * Complete: 完成
        * Deployment可用的Replica个数等于或者超过Deployment策略中期望的个数
        * 所有与该Deployment相关的Replica都更新完成
    * Failed: 失败
        * 无效的引用
        * 不可读的probe failure
        * 镜像拉取错误
        * 权限不够
        * 范围限制
        * 程序运行时配置错误
4. Deployment使用命令
    * 查询所有Deployment: $ kubectl get deployments -o wide
    * Deployment扩容: $ kubectl scale deployment nginx-deployment --replicas 10
    * 更新Deployment镜像: $ kubectl set image deployment/nginx-deployment nginx=nginx:1.9.1
    * 回滚Deployment: $ kubectl rollout undo deployment/nginx-deployment

# 七. ReplicaSet (RS)
1. 什么是ReplicaSet
    * ReplicaSet是下一代Replication Controller，ReplicaSet和Replication Controller唯一的区别是现在的选择器支持不同，官方推荐使用ReplicaSet
    * 大多数kubectl支持Replication Controller的命令也支持ReplicaSet
    * ReplicaSet主要是被Deployment做为协调Pod创建、删除和更新的机制，当使用Deployment时，你不必担心创建Pod的ReplicaSets，因为可以通过Deployment实现管理ReplicaSets
2. Deployment是一个更高层次的概念，它管理ReplicaSet，并提供对pod的声明性更新以及许多其他的功能。因此，我们建议使用Deployment而不是直接使用ReplicaSet。这实际上意味着我们可能永远不需要操作ReplicaSet对象，而是直接使用Deployment并在规范部分定义应用程序。

# 八. Replication Controller (RC)
1. 什么是Replication Controller
    * Replication Controller保证了所有时间内都有特定数量的Pod副本正在运行，和直接创建Pod不同的是Replication Controller会替换掉那些删除的或者被终止的Pod
    * 即使只创建一个Pod，我们也推荐使用Replication Controller，Replication Controller就像一个进程管理器监管着不同Node上的多个Pod，而不是单单监控一个Node上的Pod
    * Replication Controller只会对那些RestartPolicy=Always的Pod生效，Replication Controller不会去管理那些有不同启动策略的Pod
2. Replication Controller使用
    * Rescheduling 重新规划，Replication Controller能保证足够数量的Pod运行，即使节点Node失败或者挂掉的情况
    * Scaling 缩放，Replication Controller很容易控制Pod副本的数量，最简单的方式是通过修改Replicas的值
    * Rolling updates 动态更新，Replication Controller支持动态更新，当我们更新一个服务的时候，它允许我们一个一个的替换Pod

# 九. Namespace
1. 什么是Namespace
    * Namespace是一组资源和对象的抽象集合，比如可以用来将系统内部的对象划分为不同的项目组或用户组
    * 常见的Pods，Services，Replication Controllers和Deployments等都属于某一个Namespace，默认是default。而Node，PresistentVolumes等则不属于任何Namespace
    * Namespace常用于隔离不同的用户，比如K8s自带的服务一般运行在Kube-system Namespace中
2. Namespace操作
    * 查询namespace  
      $ kubectl get namespaces  
        `namespace包含两种状态Active和Terminating，删除namespace的过程中namespace状态被设置为Terminating`
    * 创建namespace  
      $ kubectl create namespace new-namespace  
        `namespace名称满足正则表达式[a-z0-9]([-a-z0-9]*[a-z0-9])?,最大长度为63位`
    * 删除namespace  
      $ kubectl delete namespaces new-namespace  
        a. 删除一个namespace会自动删除所有属于该namespace的资源  
        b. default和kube-system命名空间不可删除。

# 十. DaemonSet (DS)
1. 什么是DaemonSet
    * 一个DaemonSet可以保证所有Node都运行一个Pod
    * 当一个新的Node接入K8s集群的时候，DaemonSet会在这个Node运行一个Pod
    * 当一个Node从K8s集群剔除的时候，DaemonSet会删除该Node上的Pod
    * 当一个DaemonSet会删除的时候，所有它创建的Pod都会被清理(不一定是删除)
2. DamemonSet保证每个Node都运行一个Pod，常用来部署一些集群日志、监控或者其他系统管理的应用，常见的应用如下
    * 日志收集: fluentd，logstash等
    * 系统监控: Prometheus Node Exporter，collected，New Relic agent，Ganglia gmond等
    * 系统程序: kebu-proxy，kube-dns，glusterd，ceph等
3. 指定Node运行DaemonSet  
   DaemonSet会忽略unschedulable状态的Node，有3种方式来指定Pod只运行在指定Node节点上
    * nodeSelector: 只调度到匹配指定label的Node上
    * nodeAffnity: 功能更丰富的Node选择器，比如支持集合操作
    * podAffnity: 调度到满足条件的Pod所在的Node上

# 十一. CronJob
1. CronJob类似于Linux系统的crontab，在指定时间周期内运行指定的任务
2. CronJob创建
   ```
   apiVersion: batch/v2alpha1
   kind: CronJob
   metadata:
     name: hello
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
          
     1. .spec.schedule 指定任务的运行周期，格式同linux系统的crontab
     2. .spec.jobTemplate指定需要运行的任务，格式同Job
     3. .spec.startingDeadlineSeconds指定任务开始的截止期限
     4. .spec.concurrencyPolicy指定任务的并发策略，支持Allow、Forbid和Replace三种 
    ```  
   `$ kubectl apply -f cronjob.yaml`
3. 也可以使用kubectl run来创建CronJob  
   $ kubectl run hello --schedule="*/1 * * * *" --restart=OnFailure --image=busybox -- /bin/sh -c "date; echo Hello from the Kubernetes cluster"
4. CronJob使用
    * 获取Cronjob: $ kubectl get cronjob
    * 获取Jobs: $ kubectl get jobs
    * 删除cronjob: $ kubectl delete cronjob hello
    * 删除job: $ kubectl delete job

# 十二. Job
1. 什么是Job  
   Job负责批量处理短暂的一次性任务，即仅执行一次的任务，它保证批处理任务的一个或多个Pod成功结束
2. K8s支持以下几种Job
    * 非并行Job: 通常创建一个Pod直至其成功结束
    * 固定结束次数的Job: 设置.spec.completions创建多个Pod，直到所有Pod成功结束
    * 带有工作队列的并行Job: 设置.spec.Parallelism但不设置.spec.completions，当所有Pod结束并且至少一个成功时，Job就认为成功
3. 创建Job
   ```
   apiVersion: batch/v1
   kind: Job
   metadata:
     name: pi
   spec:
     template:
     metadata:
       name: pi
     spec:
       containers:
       - name: pi
         image: perl
         command: ["perl",  "-Mbignum=bpi", "-wle", "print bpi(2000)"]
       restartPolicy: Never
   ```
   `$ kubectl apply -f job.yaml`

# 十三. Resource Quota
1. 什么是Resource Quota  
    * Resource Quota是用来限制用户资源使用量的一种机制
    * Resource Quota应用在namespace上，并且每个namespace最多只能有一个ResourceQuota对象
    * 开启计算Resource Quota后，创建容器的时候必须配置计算资源请求或限制
    * 用户超额后禁止创建新的资源
2. Resource Quota类型
    * 计算资源
        * cpu: limits cpu, requests cpu
        * memory: limits memory, requests memory
    * 存储资源
        * requests.storage: 存储资源总量
        * persistentvolumeclaims: pvc的个数
        * .storageclass.storage.k8s.io/requests.storage
        * .storageclass.storage.k8s.io/persistentvolumeclaims
    * 对象数
        * pods, replication controllers, configmaps, secrets
        * resourcequotas, persistentvolumeclaims
        * services, services.loadbalancers, services.nodeports
3. Resource Quota类型
   ```
   计算资源示例
   apiVersion: v1
   kind: ResourceQuota
   metadata:
     name: compute-resources
   spec:
     hard:
       pods: "4"
       requests.cpu: "1"
       requests.memory: 1Gi
       limits.cpu: "2"
       limits.memory: 2Gi
   ``` 

   ```
   对象个数示例
   apiVersion: v1
   kind: ResourceQuota
   metadata:
     name: object-counts
   spec:
     hard:
       configmaps: "10"
       persistentvolumeclaims: "4"
       replicationcontrollers: "20"
       secrets: "10"
       services: "10"
       services.loadbalancers: "2"
   ```

# 十四. ConfigMap
1. ConfigMap用于保存配置数据的键值对，可以用来保存单个属性也可以用来保存配置文件，ConfigMap和secret很类似，但它可以更方便处理不包含敏感信息的字符串。
2. ConfigMap创建
    * 从key/value字符串创建ConfigMap  
      $ kubectl create configmap special-config --from-literal=special.how=very  
      $ kubectl get configmap special-config -o go-template='{{.data}}'
    * 从env文件创建  
      $ kubectl create configmap special-config --from-env-file=config.env
    * 从目录创建  
      $ mkdir config  
      $ echo a > config/a  
      $ echo b > config/b  
      $ kubectl create configmap special-config --from-file=config/
3. ConfigMap使用  
   ConfigMap可以通过多种方式在Pod中使用，比如设置环境变量、设置容器命令行参数、在Volume中创建配置文件
     * ConfigMap必须在Pod引用它之前创建
     * 使用envFrom时，将会自动忽略无效的键
     * Pod只能使用同一个namespace内的ConfigMap
4. ConfigMap使用
     * 用作环境变量
     * 用作命令行参数，将ConfigMap用作命令行参数时，需要先把ConfigMap的数据保存在环境变量中，然后通过$(VAR_NAME)的方式引用环境变量.
     * 使用Volume将ConfigMap做为文件或目录直接挂载，将创建的ConfigMap直接挂载至Pod的/etc/config目录下，其中每一个key-value键值对都会生成一个文件，key为文件名，value为内容  
5. ConfigMap使用示例
   ```
   1.先创建ConfigMap
     $ kubectl create configmap special-config --from-literal=special.how=very --from-literal=special.type=charm
     $ kubectl create configmap env-config --from-literal=log_level=INFO              
   2.YAML文件以环境变量的方式引用
   apiVersion: v1
   kind: Pod
   metadata:
     name: test-pod
   spec:
     containers:
       - name: test-container
         image: gcr.io/google_containers/busybox
         command: [ "/bin/sh", "-c", "env" ]
         env:
           - name: SPECIAL_LEVEL_KEY
             valueFrom:
               configMapKeyRef:
                 name: special-config
                 key: special.how
           - name: SPECIAL_TYPE_KEY
             valueFrom:
               configMapKeyRef:
                 name: special-config
                 key: special.type
         envFrom:
           - configMapRef:
             name: env-config
      restartPolicy: Never
                 
    3.当Pod运行结束的时候，会输出
      SPECIAL_LEVEL_KEY=very 
      SPECIAL_TYPE_KEY=charm
      log_level=INFO
   ```
   ```
   apiVersion: v1
   kind: Pod
   metadata:
     name: dapi-test-pod
   spec:
     containers:
       - name: test-container
         image: gcr.io/google_containers/busybox
         command: [ "/bin/sh", "-c", "echo $(SPECIAL_LEVEL_KEY) $(SPECIAL_TYPE_KEY)" ]
         env:
           - name: SPECIAL_LEVEL_KEY
             valueFrom:
               configMapKeyRef:
                 name: special-config
                 key: special.how
           - name: SPECIAL_TYPE_KEY
             valueFrom:
               configMapKeyRef:
                 name: special-config
                 key: special.type
     restartPolicy: Never
   ```

   ```
   apiVersion: v1
   kind: Pod
   metadata:
     name: vol-test-pod
   spec:
     containers:
       - name: test-container
         image: gcr.io/google_containers/busybox
         command: [ "/bin/sh", "-c", "cat /etc/config/special.how" ]
         volumeMounts:
           - name: config-volume
           mountPath: /etc/config
     volumes:
       - name: config-volume
         configMap:
           name: special-config
     restartPolicy: Never
   ```

# 十五. Network Policy
1. 什么是Network Policy
    * 网络策略说明一组Pod之间是如何被允许互相通信的，以及如何与其它网络Endpoint进行通信
    * Network Policy资源使用标签来选择Pod，并定义了一些规则，这些规则指明哪些流量允许进入到Pod上
2. 创建Network Policy
   ```
   apiVersion: networking.k8s.io/v1
   kind: NetworkPolicy
   metadata:
     name: test-network-policy
     namespace: default
   spec:
     podSelector:
       matchLabels:
         role: db
   ingress:
   \- from: (加个\防止被当成markdown语法)
      \- namespaceSelector:  (加个\防止被当成markdown语法)
         matchLabels:
           project: myproject
      \- podSelector:  (加个\防止被当成markdown语法)
         matchLabels:
           role: frontend
      ports:
      \- protocol: TCP  (加个\防止被当成markdown语法)
        port: 6379
   ```
   
   1). podSelector：每个NetworkPolicy 包含一个podSelector，它可以选择一组应用了网络策略的Pod。由于NetworkPolicy当前只支持定义ingress规则,这个podSelector实际上为该策略定义了一组"目标Pod"。示例中的策略选择了标签为"role=db"的Pod。一个空的podSelector选择了该Namespace中的所有Pod
这个 podSelector实际上为该策略定义了一组"目标Pod"。示例中的策略选择了标签为"role=db"的Pod。  
   2).ingress：每个NetworkPolicy包含了一个白名单ingress规则列表。每个规则只允许能够匹配上from和ports配置段的流量。示例策略包含了两个规则，它从这两个源中匹配在单个端口上的流量，第一个是通过namespaceSelector指定的，第二个是通过podSelector指定的。  

   配置说明  
   1). 在"default" Namespace中 隔离了标签"role=db"的 Pod  
   2). 在"default" Namespace中，允许任何具有"role=frontend"的Pod，连接到标签为"role=db"的Pod的TCP端口6379  
   3). 允许在Namespace中任何具有标签"project=myproject"的Pod，连接到"default" Namespace中标签为"role=db"的Pod的TCP端口6379

# 十六. StatefulSet
1. StatefulSet是为了解决有状态服务的问题，对应Deployment和ReplicaSet是为无状态服务而设计，应用场景包括
    * 稳定的持久化存储，即Pod重新调度后还能访问到相同的持久化数据
    * 稳定网络标志，即Pod重新调度后其PodName和Hostname不变
    * 有序部署，有序扩展，即Pod是有顺序的，在部署或扩展的时候要依据定义的顺序依次进行
    * 有序收缩，有序删除
2. StatefulSet使用
    * 创建: $ kubectl apply -f statefulSet.yaml
    * 查看StatefulSet: $ kubectl get statefulset
    * 扩容: $ kubectl scale statefulset web —replicas=5
    * 缩容: $ kubectl patch statefulset web -p ‘{“spec”:{“replicas”:3}}’
    * 镜像更新: $ kubectl patch statefulset web --type='json' -p='[{"op": "replace", "path": "/spec/template/spec/containers/0/image", "value":"gcr.io/google_containers/nginx-slim:0.7"}]’
    * 删除statefulSet: $ kubectl delete statefulset web

# 十七. Secret
1. Secret解决什么问题  
   Secret解决了密码、token、秘钥等敏感数据的配置问题，不需要把这些敏感数据暴露到镜像或者Pod Spec中，Secret可以以Volume或者环境变量的方式使用，Secret有以下3种类型
    * Service Account: 用来访问K8s API，由K8s自动创建，并且会自动挂载到Pod的/run/secrets/kubernetes.io/serviceaccount目录
    * Opaque: base64编码格式的Secret，用来存储密码、秘钥
    * kubernets.io/dockerconfigjson: 用来存储私有docker register的认证信息
2. 创建Secret
   ```
   apiVersion: v1
   kind: Secret
   metadata:
    name: mysecret
   type: Opaque
   data:
    password: MWYyZDFlMmU2N2Rm
    username: YWRtaW4=
   ```
3. 将Secret导出到环境变量
   ```
   apiVersion: extensions/v1beta1
   kind: Deployment
   metadata:
     name: wordpress-deployment
   spec:
     replicas: 2
     strategy:
       type: RollingUpdate
     template:
       metadata:
         labels:
           app: wordpress
           visualize: "true"
       spec:
         containers:
         - name: "wordpress"
           image: "wordpress"
           ports:
           - containerPort: 80
           env:
           - name: WORDPRESS_DB_USER
             valueFrom:
               secretKeyRef:
                 name: mysecret
                 key: username
           - name: WORDPRESS_DB_PASSWORD
             valueFrom:
               secretKeyRef:
                 name: mysecret
                 key: password
   ```
   
