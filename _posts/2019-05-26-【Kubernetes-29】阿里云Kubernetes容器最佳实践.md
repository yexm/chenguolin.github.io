---
layout:  post  # 使用的布局（不需要改）
catalog: true  # 是否归档
author: 陈国林 # 作者
tags:          #标签
    - Kubernetes
---

本文参考自 [阿里云容器产品实践.pdf](https://github.com/chenguolin/chenguolin.github.io/blob/master/data/pdf/%E9%98%BF%E9%87%8C%E4%BA%91%E5%AE%B9%E5%99%A8%E4%BA%A7%E5%93%81%E5%AE%9E%E8%B7%B5.pdf)

# 一. 容器平台构建
1. **集群容量规划**  
   服务总实例 * 实例规格 * 冗余，例如有18个微服务每个服务2个实例，每个实例用2c/4g，预留10%冗余  
   总资源:18 X 2 X 2c/4g X 110% = 72c/144g X 110% = 80c/160g   
   注意:需要估算daemonset, job等类型的资源

2. **集群网络规划**  
   明确ECS所在的VPC的IP段，是否与其它服务关联(RDS)，是否有专线与私有云IDC互联，路由是否已经打通等   
   明确POD的IP段，是否需要与非K8S集群的ECS互联，例如其它ECS上部署的服务 
   明确Service的IP段，此IP为虚IP不可与其它ECS的IP段冲突  
   需要考虑跨可用区的网络设计

3. **集群ECS选型**  
   Master机型的建议选择:100节点以内8c/32g、100~200节点16c/64g，尽量预留多点资源
   Woker机型的选择:根据总体容量以及可以容忍的ECS掉线率来计算使用的规格，如1000c的总容量，那么容忍10%的容量掉线率，那么可以使用10台96c的神龙机器  
   Woker容量预留:确保剩余的可用容量能承担任意一台ECS掉线带来的容器漂移  
   
   `为什么Worker尽可能选高规格的机器？`
   + Woker数量相对少了，Master的负担最小，调度性能最优，减轻了运维的维护工作量和成本
   + Woker数量相对少了，Pull image的次数减少，减少带宽占用，提高容器调度效率
   + Woker数量相对少了，剩余碎片资源少，资源利用率最好（同系列规格机器，高规格机器没有增加成本，反而因为系统盘减少而相对减少成本）
   + Woker单机规格增大，驱逐的概率降低

4. **集群网络选型**  
   支持两种模型:flannel和terway，terway具备更高的性能以及支持 network policy  
   注意选择节点支持的POD的IP个数，避免IP浪费，从而影响集群能支持的节点的个数  
   支持最大节点数:POD的最大IP数/每个节点POD的IP个数(与节点支持的POD个数无关--max-pods)

5. **集群配置设置**  
   Worker配置: 需要选择挂载数据盘，该数据盘用在 /var/lib/docker 以及 /var/lib/kubelet，避免镜像多次拉取后撑爆系统盘（系统盘和数据盘）  
   确保机器都可以上网(配置SNAT)  
   磁盘的空间不要设置太小，生产一般给系统盘留`100G`，尽量使用SSD云盘

# 二. 容器平台运维
1. **多集群与Namespace**  
   集群区分环境(开发/测试/生产)，Namespace用于区分不同的业务领域  

2. **独立部署Ingress Controller**  
   选用神龙32c/64g的配置2~3台对应一个SLB(5G入口流量)   
   为部署的机器打上污点标签例如Taint: Ingress=:NoEexcute  
   设置Ingress controller的POD资源限制为request/limit: 16c/32g  
   设置Ingress controller使用ENI  
   设置对应的Nginx的配置的worker数为16  
   更改对应的SLB个规格(支持的并发数)，以及选择按照流量计费 (最大支持5G)  

3. **多集群kubectl管理适配**  
   通过kubectl config view 查看集群配置  
   配置环境变量KUBECONFIG，使用KUBECONFIG环境变量配置多个集群的config文件  
   通过kubectl config get-contexts获取对应的集群信息，通过kubectl config user-context <context name>切换集群  
   对于子账户，可以无缝使用config合并方式操作，对于主账户，需要做下config文件修改，因为默认都使用了 kubernetes-admin 作为用户，kubernetes作为集群名字，kubernetes-admin@kubernetes作为context名字  
    
4. **kubectl工具集**  
   集群切换、命名空间切换工具kubectlx、kubens: `https://github.com/ahmetb/kubectx`  
   Kubectl短命令集: `https://github.com/ahmetb/kubectl-aliases`  
   命令行展示Kubernetes的集群名称和NS名称: `https://github.com/jonmosco/kube-ps1`  
   K8S资源跟踪工具kubespy: `https://github.com/pulumi/kubespy`  
   Kubectl插件管理工具krew: `https://github.com/GoogleContainerTools/krew`  
   同时查看多个POD中多个容器的log: `https://github.com/wercker/stern`  
   Docker Image分析工具: `https://github.com/wagoodman/dive`  

5. **Image使用**  
   同一个版本的同一个应用确保只有一个Image，不要因为环境的不同，而使用不同的镜像  
   配置信息务必脱离镜像  
   继承的父镜像务必标示tag，不可以用latest作为生产镜像  
   镜像务必来源与同一个镜像仓库  
   镜像务必经过安全扫描  
   镜像只由Dockerfile构建，使用多阶段构建，确保镜像最干净/最小
   
6. **Master控制**  
   默认情况下，Master不会运行业务的POD，除非设置对应的容忍  
   在测试环境下，可以去除Master的taint来运行对应的业务POD: `kubectl taint nodes --all node-role.kubernetes.io/master-`，如果需要恢复，则可以补充上这个taint，不过需要注意的是已经运行的POD是不会被驱逐的。注意不要给node节点打上这个taint

7. **存储使用**  
   对于需要POD消亡后保留数据，使用PV的方式，不可直接用本地磁盘Volume  
   对于同一个StatefulSet每个POD需要自己的PV，可以使用云盘的存储  
   对于同一个Deployment每个POD需要共享存储用NAS，同时对于读/写都很频繁的场景也要用NAS  
   对于一次写入(少数写入)，然后多次读的场景，可以使用OSS  

8. **POD优先级**  
   1.11.x开始，Pod优先级特性默认开启，默认优先级为0，最低优先级  
   优先级资源priorityClass是全局的  
   默认提供两个系统级的priorityClass给critical pod  
   优先级会启用抢占特性(scheduler开关disablePreemption控制，默认false)  

9. **容器伸缩**  
   HPA伸缩
   Cluster Autoscaler  
   
10. **查错步骤**  
   ECS互通是否有异常  
   安全组是否配置有误  
   使用子账户，授权是否正确  
   Docker run的方式运行，是否能正常提供服务  
   在K8S里通过一个POD去访问目标的POD是否正常  
   通过service方式访问是否正常  
   通过SLB/Ingress方式访问是否正常   
   Kubectl get event是否有异常  
   Api server/schedule/controller日志是否有异常  
   Docker daemon日志是否有异常  
   
11. **日志查看**  
   kubectl describe xxx   
   Docker引擎日志: `journalctl -u docker -f`   
   Kubelet日志: `journalctl -u kubelet -f`  
   API Server日志: `docker logs <api server container id>`  
   Scheduler日志: `docker logs <scheduler container id>`  
   Master proxy日志: `docker logs <master proxy container id>`  
   Worker proxy日志: `docker logs <worker proxy container id>`  
   Controller日志: `docker logs <controller container id>` 
   
12. **容器和主机时区一致**  
   方式一:通过TZ环境变量设置时区 `TZ=Asia/Shanghai`   
   方式二:通过挂载主机的时区文件  
 
13. **taint的三种效果**  
   Noschedule: 不调度没有toleration的新pod到该节点  
   PreferNoSchedule: 不优先调度的节点  
   NoExecute: 驱逐不带toleration的pod  

# 三. 容器平台配置管理
1. **配置原则**  
   配置与镜像务必严格分离  
   区分清楚初始化配置还是需要运行时可改动配置  
   初始化配置放到configmap以及secret上  
   动态配置通过配置中心(如阿里云的ACM)配置和推送   
   不影响兼容性的新功能特性(feature)务必配置动态开关，确保可以实时关闭，避免新功能错误带来的影响  
   影响业务性能的特性务必配置动态开关，确保可以实时关闭，避免业务高峰无法支撑的影响  
   务必通过灰度方式打开动态的配置开关  
   配置一定要做到有迹可循  
   
2. **configmap/secret**  
   可以通过Env或者Volume方式使用  
   配置变更了，POD的Env是不会发生变化的  
   配置变更了，Volume的文件在POD是会发生变更的，但是变更生效时间不可控  
   无论那种方式，都建议用POD rolling upgrade的方式来获取新的配置  
   Configmap/secret映射到容器文件都是只读的  

# 三. 容器平台开发管理
1. **容器安全**  
   严格约束使用主机IPC, PID namespace选项(hostPID, hostIPC)  
   避免使用root用户运行，设置securityContext来设置运行用户(runAsUser)或者是设置为非root用户(runAsNonRoot)  
   严格约束使用特权模式(privileged)  
   根据特定需求来增加/删除细粒度的容器能力  
   根据需要禁止写入容器的本地文件系统  
   使用PodSecurityPolicy/Pod Preset来简化运维  

2. **容器运行**  
   Namespace的资源受quota控制  
   每个pod都应该设置limit, request  
   通过limitRange来配置默认limit, request  

3. **健康检查**  
   k8s健康检查双保险: liveness和readiness  
   Readiness: 确保应用已经可以接收流量了，不然不会分发流量给POD  
   Liveness: 确保应用还存活，不然就会重启这个POD （用于检查的三种类型探针:http, cmd, tcp） 

4. **优雅退出**  
   容器中的主进程(pid=1)将会收到SIGTERM，然后需要处理相应的程序内资源释放处理，时间由terminationGracePeriodSeconds控制  
   利用好preStop hook，执行对应的进程外清理操作(没有参数传递)， 这个设置执行前，terminationGracePeriodSeconds已经开始计算  
   Dockerfile中使用CMD ["/bin/bash", "-c", "myapp -- arg=$ENV_VAR"]格式启动程序，避免SIGTERM不被传递问题(因为 CMD xxx，实际是/bin/sh –c xxx，在某些Linux下会不传递 SIGTERM)
   
