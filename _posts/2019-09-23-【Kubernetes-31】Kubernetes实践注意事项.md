---
layout:  post  # 使用的布局（不需要改）
catalog: true  # 是否归档
author: 陈国林 # 作者
tags:          #标签
    - Kubernetes
---

# 注意事项
1. 如果容器1号进程是shell脚本，应用进程是这个shell进程的子进程，那这个shell进程是收不到 SIGTERM 信号的
   ```
   这种启动方式是Pod配置文件container启动参数的shell脚本，例如 "/bin/sh /run.sh -c ..."，这种方式业务应用进程是通过run.sh脚本启动
   在容器内部，PID为1进程为run.sh，业务进程为1号进程子进程
   
   这种方式有个问题就是，容器内1号是收不到操作系统 SIGTERM 信号的，具体可以参考
   
   Note: A process running as PID 1 inside a container is treated specially by Linux: it ignores any signal with the default action. 
   So, the process will not terminate on SIGINT or SIGTERM unless it is coded to do so.
   
   One mistake which is pretty common and something which can be missed is using the non-exec form of CMD.
   for example, then your process is running as a child process of shell and not really running as root. It will be running as /bin/sh -c myapplication
   
   The problem with this is that, shell will never forward this signal to the child process, which is your application process and it will not be able to handle the SIGTERM for which you have written a handler for.
   
   In any case, this would make your process be SIGKILLed. Instead one should use the exec form CMD
   ```

2. 容器Pod一直处于Blocked状态，通过describe命令发现Pod有 `forbidden sysctl: "net.core.somaxconn" not whitelisted` 这个信息
   ```
   原因是kubelet没有开启unsafe机制，而我们又在Pod yaml配置文件里面定义了以下配置
   securityContext:
        sysctls:
        - name: net.core.somaxconn
          value: "10000"
          
   解决方案是: 可以先把securityContext配置去掉，但是这个配置会影响整个Pod内容器最大链接数，默认为128，明显是不够。
   另外我们需要在kubelet上把对应配置开起来，保证配置可以验证通过，不至于kubelet fail fast
   ```
   可以参考官方文档: https://kubernetes.io/docs/tasks/administer-cluster/sysctl-cluster/  
   里面有提到 `All unsafe sysctls are disabled by default and must be allowed manually by the cluster admin on a per-node basis. Pods with disabled unsafe sysctls will be scheduled, but will fail to launch`

3. 一般情况下如果我们希望某些业务跑在特定的节点上，同时不希望其它业务跑在这个节点上，一般我们的做法是这样的
    + 对特定的节点打标签，同时打上污点。打标签是为了做节点选择，打污点则是为了保证其它业务不会调度上来。
    + 业务Pod新增nodeSelector只选择调度在特定标签的节点上，同时容忍特定的污点保证能够调度成功。
 
4. K8s Pod在启动的时候一直处于ContainerCreating的状态，查看它的events发现有这个错误 ` container init caused \"read init-p: connection reset by peer\"": unknown`，最后确认是因为 `发现是因为容器资源Limit设置的太小了，导致进程无法起来`。 
    + CPU的设置可以是 `1.0` 等价于 `1000m` 或者 `0.1` 等价于 `100m`
    + Mem的设置可以是 `128974848` 等价于 `129e6` 等价于 `129M` 等价于 `123Mi`

5. K8s Deployment、Daemonset 使用 RollingUpdate 进行滚动更新的话，如果变更了 labels 那么会报错 ```spec.template.metadata.labels: Invalid value: map[string]string{"k8s-app":"app", "kubernetes.io/cluster-service":"true"}: `selector` does not match template `labels` ```，这个时候解决方案是先使用 kubectl edit 方式编辑yaml配置文件，去掉 matchLabels 对应的Key-Value，再使用RollingUpdate进行滚动升级。

6. ephemeral-storage 本地临时存储容器限制，在最新的k8s版本 v1.16 中还是beta特性。使用 ephemeral-storage 需要注意的是 它只能限制 `/` 跟分区，跟分区包括 /var/lib/kubelet、/var/log 等这些目录。因此，需要注意的是 emptyDir 会被 ephemeral-storage 所约束，否则会出现 Pod 因为使用超过了指定的存储空间被驱逐了。

7. K8s中关于内存单位Mi和M的区别，Mi表示`1Mi=1024x1024`，M表示`1M=1000x1000`，其它单位类似（Ki与K，Gi与G）

8. K8s Pod在起来的时候容器经常会报以下的错误
   ```
   1). standard_init_linux.go:207: exec user process caused "no such file or directory"
       原因: entrypoint脚本设置的shabang错误 应该是 /bin/sh
   2). standard_init_linux.go:207: exec user process caused "exec format error"
       原因: shell脚本有不符合linux规范的字符，例如中文等
   ```

9. K8s Pod的一些属性限制 (强烈建议K8s相关的API对象属性值不超过63字符，否则因为K8s限制会报错) [identifiers-and-names-in-kubernetes](https://github.com/kubernetes/community/blob/master/contributors/design-proposals/architecture/identifiers.md#identifiers-and-names-in-kubernetes)
    + label的值不能超过`63`个字符，否则会报 `must be no more than 63 characters` 
    + container的name字段值不能超过`63`个字符，否则会报 `must be no more than 63 characters` 

10. 使用Go开发一个K8s相关组件的时候，经常会遇到一个K8s相关组件依赖理不清的情况。正常情况下，我们主要的依赖的组件是 `k8s.io/client-go`、`k8s.io/api`、`k8s.io/apimachinery`。如果我们直接依赖了 `k8s.io/kubernetes` 这个项目，很大概率会导致 dep 在更新依赖的时候报错。
    ```
    报错内容类似
    ...
    Solving failure: Could not introduce k8s.io/kubernetes@v1.17.0, as it requires package k8s.io/client-go/tools/events from     k8s.io/client-go, but in version kubernetes-1.12.0 that package is missing.
    ...
    
    解决方案
    1). 删除Gopkg.toml、Gopkg.lock、vendor，使用 dep init -v 命令重新生成依赖
    2). 如果实在无法解决，尽量避免直接依赖 k8s.io/kubernetes 把用到的相关代码 copy 到项目的pkg下，不过这有个问题就是后续如果有代码的变更就没有办法更新了，必须自己手动更新
    ```

11. kubernetes集群搭建的时候涉及到很多国外的镜像，但是由于机器在国内导致很多镜像都拉取不到，因此需要做些调整，先从国内的镜像源拉取，然后使用 docker tag 重新打下tag
    ```
    1. docker.io 相关的镜像，可以使用 dockerhub.azk8s.cn 代替，下载完成之后retag即可
    2. gcr.io 相关的镜像，可以使用 gcr.azk8s.cn 代替，下载完成之后retag即可
    3. quay.io 相关的镜像，可以使用 quay.azk8s.cn 代替，下载完成之后retag即可
    ```

