---
layout:  post  # 使用的布局（不需要改）
catalog: true  # 是否归档
author: 陈国林 # 作者
tags:          #标签
    - Kubernetes
---
# 一. Kubectl
1. `kubectl annotate` 更新资源的注解，支持的资源包括但不限于（大小写不限）pods (po)、services (svc)、 replicationcontrollers (rc)、nodes (no)、events (ev)、componentstatuses (cs)、 limitranges (limits)、persistentvolumes (pv)、persistentvolumeclaims (pvc)、 resourcequotas (quota)和secrets。
2. `kubectl api-versions` 以“组/版本”的格式输出服务端支持的API版本。
3. `kubectl apply` 通过文件名或控制台输入，对资源进行配置，接受JSON或者YAML格式的描述文件
4. `kubectl attach` 连接到一个正在运行的容器
5. `kubectl autoscale` 对replication controller进行自动伸缩
6. `kubectl cluster-info` 输出集群信息
7. `kubectl config` 修改kubeconfig配置文件
8. `kubectl create` 通过文件名或控制台输入，创建资源
9. `kubectl delete` 通过文件名、控制台输入、资源名或者label selector删除资源
10. `kubectl describe` 输出指定的一个/多个资源的详细信息
11. `kubectl edit` 编辑服务端的资源
12. `kubectl exec` 在容器内部执行命令
13. `kubectl expose` 输入replication controller，service或者pod，并将其暴露为新的kubernetes service
14. `kubectl get` 输出一个/多个资源
15. `kubectl label` 更新资源的label
16. `kubectl logs` 输出pod中一个容器的日志
17. `kubectl namespace` （已停用）设置或查看当前使用的namespace
18. `kubectl patch` 通过控制台输入更新资源中的字段
19. `kubectl port-forward` 将本地端口转发到Pod
20. `kubectl proxy` 为Kubernetes API server启动代理服务器
21. `kubectl replace` 通过文件名或控制台输入替换资源
22. `kubectl rolling-update` 对指定的replication controller执行滚动升级
23. `kubectl run` 在集群中使用指定镜像启动容器
24. `kubectl scale` 为replication controller设置新的副本数
25. `kubectl stop`（已停用）通过资源名或控制台输入安全删除资源
26. `kubectl version` 输出服务端和客户端的版本信息

# 二. 问题排查
## ① 常用命令
1. 列出所有namespace下所有Container  
   `$ kubectl get pods --all-namespaces -o jsonpath="{..image}” | \`  
     `tr -s '[[:space:]]' '\n’ | \`  
     `sort | \`  
     `uniq -c`
2. 进入Pod执行/bin/bash  
   `$ kubectl exec test-output-b7b965464-xq8s8 -it -n kube-system /bin/bash`  
   `$ kubectl exec test-output-b7b965464-xq8s8 -c ruby-container -it -n kube-system /bin/bash  (-c 指定某个container)`
3. 重启Depolyment里面的Pod  
   `$ kubectl delete pods <pod_name> -n namespace （如果只有单个Pod，删除Pod之后Deployment会自动拉取一个新的Pod）`
4. 删除某个DaemonSet  
   `$ kubectl delete ds fluentd -n kube-system`
5. 查看node状态  
   `$ kubectl get nodes`
6. 查看某个namespace的events  
   `$ kubectl get events -n kuby-system`
7. 查看DaemonSet滚动升级的进度  
   `$ kubectl rollout status ds fluentd -n kube-system`
8. 强制删除Pod或者DaemonSet  
   `$  kubectl delete --force --grace-period=0 pod/fluentd-xxxxx -n kube-system (多个pod之间用空格分开) `  
   `$  kubectl delete --force --grace-period=0 ds/fluentd -n kube-system`
9. 获取当前集群apiserver地址  
   `$ (kubectl config view | grep server | cut -f 2- -d ":" | tr -d " “)`
10. 获取集群认证token  
   `$ TOKEN=$(kubectl describe secret $(kubectl get secrets | grep default | cut -f1 -d ' ') | grep -E '^token' | cut -f2 -d':' | tr -d '\t’)`
11. curl请求k8s集群apiserver  
   `$ curl $APISERVER/api --header "Authorization: Bearer $TOKEN" —insecure`
12. kubeconfig配置说明  
    `1). --certificate-authority: 集群公钥`  
    `2). --client-certification: 客户端证书`   
    `3). --client-key: 客户端私钥`  
       注意: 客户端的证书需要使用集群公钥也就是--certificate-authority签署，否则是不会被集群认可的  
       还可以使用token认证的方式与apiserver进行通信
13. 查询某些pod的pod id  
    `$ kubectl get all -n mt-xiuxiu -o wide | grep 'po/xiuxiu-php-' | grep 9596c5d6d | cut -d' ' -f1 | grep po | xargs -I {} kubectl get {} -n mt-xiuxiu -o yaml | grep uid | grep -v deb3363a`
14. 查看emptydir Volume在node上的目录  
    1). 找到pod: `$ kubectl get po -n cgltest -o wide | grep k8s-log`   
    2). 查看pod的node ip: `$ kubectl get po k8s-log-7777bbbbcc-9wp6r -n cgltest -o yaml | grep hostIP`  
    3). 查看pod id: `$ kubectl get po k8s-log-7777bbbbcc-9wp6r -n cgltest -o yaml | grep uid`  
    4). 登录node 查看 /var/lib/kubelet/pods/{pod_id}/volumes/kubernetes.io~empty-dir/ 目录即可
15. 查看所有Namespace下的deployment的yaml  
   `$ kubectl get deployment --all-namespaces -o yaml`
16. 通过kubeconfig访问特定的k8s集群  
   `$ kubectl get nodes --kubeconfig kubeconfig`
17. 查看node对应的污点  
   ```$ kubectl get nodes -o go-template='{{printf "%-50s %-12s\n" "Node" "Taint"}}{{- range .items}}{{- if $taint := (index .spec "taints") }}{{- .metadata.name }}{{ "\t" }}{{- range $taint }}{{- .key }}={{ .value }}:{{ .effect }}{{ "\t" }}{{- end }}{{- "\n" }}{{- end}}{{- end}}'```
18. daemonset滚动更新  
   1). 更新yaml配置文件: `$ kubectl apply -f daemonset.yaml`  
   2). 编辑更新策略为: `$ kubectl edit ds -n kube-system fluentd   (updateStratege.type为RollingUpdate)`   
   3). 查看更新状态: `$ kubectl rollout status ds fluentd -n kube-system`
18. 查看pod内某个容器的日志  
   `$ kubectl logs -f -n kube-system {pod-name} -c {container-name}`
19. 删除namespace，将会删除该ns下所有pod  
   `$ kubectl delete ns my-namespace`
20. 拷贝文件到K8s Pod  
   1). 拷贝单文件到Pod: `$ kubectl cp [file-path] [pod-name]:/[path]` 例如 `$ kubectl cp sample.dat myapp-759598b9f7-7gbsc:/tmp/`   
   2). 拷贝目录到Pod: `$ kubectl cp [dir-path] [pod-name]:/[path]` 例如 `$ kubectl cp dir myapp-759598b9f7-7gbsc:/tmp/`  
   3). 拷贝文件到多个Pod: `$ for podname in $(kubectl get pods -n test -o json | jq -r '.items[].metadata.name' | grep cgl-test); do kubectl cp log ${podname}:/tmp/ -n test ; done`   
   4). 确认文件是否拷贝成功: `$ for podname in $(kubectl get pods -n test -o json | jq -r '.items[].metadata.name' | grep cgl-test); do kubectl exec -i ${podname} -n test ls /tmp ; done`   
21. 删除污点  
   `$ kubectl taint nodes {node-name} k8s.toBeDeleted:NoSchedule-`  //注意末尾有个 `-`
22. 列出Pod名称长度超过40  
   `$ kubectl get all --all-namespaces -o name | awk -F '/' '{print $2}' | awk '{if (length($1) > 40) print}'`

## ② 常用问题排查步骤
1. 查看Pod状态以及运行的节点: `$ kubectl get pods -n kube-system -o wide`
2. 查看Pod事件: `$ kubectl describe pod {pod_name} -n kube-system` 或 `$ kubectl get event -n kube-system --field-selector involvedObject.name=my-pod-zl6m6`
3. 查看Events: `$ kubectl get events -n kube-system` 
4. 查看Node状态: `$ kubectl get nodes` 或 `$ kubectl describe node {node_name}`
5. 查看kubelet日志: `$ journalctl -l -u kubelet   (Kubelet通常以 systemd 管理)`
6. 查看dockerd日志: `$ grep dockerd /var/log/messages`
7. 查看containerd日志: `$ grep containerd /var/log/messages`
8. 查看containerd-shim日志: `$ grep containerd-shim /var/log/messages`
9. 查看docker-runc日志: `$ grep docker-runc /var/log/messages`
10. 查看kube-apiserver日志: `$ journalctl -l -u kube-apiserver`
11. 查看kube-controller-manager日志: `$ journalctl -l -u kube-controller-manager`
12. 查看kube-scheduler日志: `$ journalctl -l -u kube-scheduler`
13. 查看etcd日志: `$ journalctl -l -u etcd`

## ③ 相关目录
1. Linux容器使用device mapper做为storage driver
   ```
   1). rootfs(镜像)挂载点文件: /var/lib/docker/image/devicemapper/layerdb/mounts/{container_id}/mount-id
   2). rootfs挂载点目录: /var/lib/docker/devicemapper/mnt/{mount_id}/rootfs
   3). 容器输出stdout/stderr日志目录: /var/log/containers
   4). 容器cgroup配置目录: /sys/fs/cgroup/{resource}/docker/{container_id}
   5). 容器配置文件目录: /var/lib/docker/containers/{container_id}/config.v2.json
   6). docker containerd目录: /run/docker/containerd/{container_id}
   7). docker runc状态目录: /run/docker/runtime-runc/moby/{container_id}/state.json
   8). docker runtime目录: /run/docker/containerd/daemon/io.containerd.runtime.v1.linux/moby/{container_id}
   9). emptyDir Volume在Node上目录: /var/lib/kubelet/pods/{pod_id}/volumes/kubernetes.io~empty-dir/
   ```
2. Linux容器使用overlay2做为storage driver
   ```
   1). rootfs(镜像)挂载点文件: /var/lib/docker/image/overlay2/layerdb/mounts/{container_id}/mount-id
   2). rootfs挂载点目录: /var/lib/docker/overlay2/{mount_id}/merged
   3). 容器输出stdout/stderr日志目录: /var/log/containers
   4). 容器cgroup配置目录: /sys/fs/cgroup/{resource}/podruntime/docker/kubepods/{container_id}
   5). 容器配置文件目录: /var/lib/docker/containers/{container_id}/config.v2.json
   6). docker containerd目录: /run/desktop/docker/containerd/{container_id}
   7). docker runc状态目录: /run/desktop/docker/runtime-runc/moby/{container_id}/state.json
   8). docker runtime目录: /run/desktop/docker/containerd/daemon/io.containerd.runtime.v1.linux/moby/{container_id}
   9). emptyDir Volume在Node上目录: /var/lib/kubelet/pods/{pod_id}/volumes/kubernetes.io~empty-dir/
   ```
