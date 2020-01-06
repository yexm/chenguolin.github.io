---
layout:  post  # 使用的布局（不需要改）
catalog: true  # 是否归档
author: 陈国林 # 作者
tags:          #标签
    - Kubernetes
---

# 一. 背景
通过前面几篇文章总结下来，我们大致的了解了 Kubernetes 的基础原理和架构，以及它所提供的API对象和 Kubernetes本质即容器编排。和其它的项目一样，我们要想真正的学习和了解它的话，必须要学会部署和安装。因此，这篇文章会介绍下如何部署一个生产可用的Kubernetes集群，整个部署会包含以下几个部分。

Kubernetes使用Go语言开发，已经免去了类似Python需要按照语言级别包依赖的麻烦，但是我们知道Kubernetes架构很复杂，需要做很多的运维工作才能够成功的把Kubernetes集群搭建起来。目前比较大的公司或云厂商一般是使用 SaltStack、Ansible 等运维工具自动化的来完成这些步骤，但是为了能够更好的了解Kubernetes，本文会采用 [kubeadm](https://github.com/kubernetes/kubeadm) 来搭建Kubernetes集群。

# 二. 搭建
## ① 前置准备
为了方便测试，我们的集群规模设定为 `1个master节点+2个worker节点`，因此我们需要准备好3个台机器，当然这个机器可以是物理机或者是虚拟机，最快的方法是通过公有云厂商直接申请 ECS 服务器。这些机器需要满足以下几个条件

1. `能够安装Docker`: 我们选择Docker做为容器运行时环境，因此要求机器是 64 位的 Linux 操作系统，同时内核版本在 3.10 及以上
2. `机器之间网络互通`: 节点必须要能够互通网络
3. `能够访问外网`: 节点会需要安装相关的rpm包，同时worker节点需要拉取镜像，因此需要能够访问外网
4. `2CPU、4GB内存`: 为了能够保证有足够的资源调度Pod，机器规格至少是 2CPU、4GB内存
5. `40GB磁盘空间`: worker节点需要存储Docker镜像和容器日志，因此至少需要有40GB的磁盘空间

我这边使用某个云厂商申请了3台机器，机器规格如下

![](https://github.com/chenguolin/chenguolin.github.io/blob/master/data/image/kubernetes-deploy-node.png?raw=true)

## ② 配置master
1. 安装Docker
   ```
   $ yum install -y yum-utils    //安装依赖
   $ yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo   //add docker-ce.repo
   $ yum list docker-ce --showduplicates  //查看所有的docker-ce rpm包版本
   $ yum install -y docker-ce-18.03.1.ce-1.el7.centos  //安装18.03.1.ce-1.el7.centos这个版本
   $ docker version   //确认docker安装成功
   Client:
    Version:      18.03.1-ce
    API version:  1.37
    Go version:   go1.9.5
    Git commit:   9ee9f40
    Built:        Thu Apr 26 07:20:16 2018
    OS/Arch:      linux/amd64
    Experimental: false
    Orchestrator: swarm
   ```
   
2. 配置 /etc/docker/daemon.json (Linux默认存储位置，如果没有创建一个新文件)
   ```
   {
	"authorization-plugins": [],
	"data-root": "",
	"dns": [],
	"dns-opts": [],
	"dns-search": [],
	"exec-opts": [],
	"exec-root": "",
	"experimental": true,
	"features": {},
	"storage-driver": "overlay2",
	"storage-opts": [],
	"labels": [],
	"live-restore": true,
	"log-driver": "json-file",
	"log-opts": {
		"max-size": "100m",
		"max-file":"5",
		"labels": "somelabel",
		"env": "os,customer"
	},
	"mtu": 0,
	"pidfile": "",
	"cluster-store": "",
	"cluster-store-opts": {},
	"cluster-advertise": "",
	"max-concurrent-downloads": 3,
	"max-concurrent-uploads": 5,
	"default-shm-size": "64M",
	"shutdown-timeout": 15,
	"debug": true,
	"hosts": ["unix:///var/run/docker.sock"],
	"log-level": "",
	"swarm-default-advertise-addr": "",
	"api-cors-header": "",
	"selinux-enabled": false,
	"userns-remap": "",
	"group": "",
	"cgroup-parent": "",
	"default-ulimits": {
		"nofile": {
			"Name": "nofile",
			"Hard": 64000,
			"Soft": 64000
		}
	},
	"init": false,
	"init-path": "/usr/libexec/docker-init",
	"ipv6": false,
	"iptables": true,
	"ip-forward": false,
	"ip-masq": false,
	"userland-proxy": false,
	"userland-proxy-path": "/usr/libexec/docker-proxy",
	"ip": "0.0.0.0",
	"bridge": "",
	"bip": "",
	"fixed-cidr": "",
	"fixed-cidr-v6": "",
	"default-gateway": "",
	"default-gateway-v6": "",
	"icc": false,
	"raw-logs": false,
	"allow-nondistributable-artifacts": [],
	"registry-mirrors": [],
	"seccomp-profile": "",
	"insecure-registries": [],
	"no-new-privileges": false,
	"default-runtime": "runc",
	"oom-score-adjust": -500,
	"node-generic-resources": ["NVIDIA-GPU=UUID1", "NVIDIA-GPU=UUID2"],
	"runtimes": {
		"cc-runtime": {
			"path": "/usr/bin/cc-runtime"
		},
		"custom": {
			"path": "/usr/local/bin/my-runc-replacement",
			"runtimeArgs": [
				"--debug"
			]
		}
	},
	"default-address-pools":[
		{"base":"172.80.0.0/16","size":24},
		{"base":"172.90.0.0/16","size":24}
	]
   }
   ```

3. 使用 systemd 启动 docker  (Linux发行版大都支持systemd启动后台常驻进程)
   ```
   $ touch /etc/systemd/system/docker.service.d/docker.conf   //如果没有创建一个新文件
   $ vi /etc/systemd/system/docker.service.d/docker.conf
     [Service]
     ExecStart=
     ExecStart=/usr/bin/dockerd
   $ systemctl daemon-reload
   $ systemctl start docker.service
   $ ps axu | grep dockerd
     root      7217  0.3  1.8 858796 71964 ?        Ssl  08:51   0:00 /usr/bin/dockerd
   $ docker info
     
   ```


2. 启动Docker daemon
   ```
   $ 
   
   $ ps axu | grep dockerd
     root      9381  0.5  1.5 333692 59836 pts/0    Sl+  14:14   0:00 dockerd --data-root=/var/lib/docker --log-opt max-size=100m --log-opt max-file=5 --iptables=false --experimental=true --storage-driver overlay2
   
   $ docker info
   Containers: 0
    Running: 0
    Paused: 0
    Stopped: 0
   Images: 0
   Server Version: 18.03.1-ce
   Storage Driver: overlay2
   ...
   ```

3. 配置Kubernetes yum repo
   ```
   $ cat <<EOF > /etc/yum.repos.d/kubernetes.repo
   [kubernetes]
   name=Kubernetes
   baseurl=http://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64
   enabled=1
   gpgcheck=0
   repo_gpgcheck=0
   gpgkey=http://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg
          http://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
   EOF
   
   注意: Kubernetes官网给的yum源是packages.cloud.google.com，但国内访问不了，我们使用阿里云的yum仓库镜像来代替
   ```

4. 安装kubeadm
   ```
   $ yum install -y kubeadm
   $ kubeadm

    ┌──────────────────────────────────────────────────────────┐
    │ KUBEADM                                                  │
    │ Easily bootstrap a secure Kubernetes cluster             │
    │                                                          │
    │ Please give us feedback at:                              │
    │ https://github.com/kubernetes/kubeadm/issues             │
    └──────────────────────────────────────────────────────────┘

    Example usage:

    Create a two-machine cluster with one control-plane node
    (which controls the cluster), and one worker node
    (where your workloads, like Pods and Deployments run).
    
    ......
   ```
   
5. 

## ③ 配置worker
1. 安装Docker
   ```
   $ yum install -y yum-utils    //安装依赖
  
   $ yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo   //add docker-ce.repo
   
   $ yum list docker-ce --showduplicates  //查看所有的docker-ce rpm包版本
   
   $ yum install docker-ce-18.03.1.ce-1.el7.centos  //安装18.03.1.ce-1.el7.centos这个版本
   
   $ docker version   //确认docker安装成功
   Client:
    Version:      18.03.1-ce
    API version:  1.37
    Go version:   go1.9.5
    Git commit:   9ee9f40
    Built:        Thu Apr 26 07:20:16 2018
    OS/Arch:      linux/amd64
    Experimental: false
    Orchestrator: swarm
   ```

2. 启动Docker daemon
   ```
   $ dockerd --data-root=/var/lib/docker --log-opt max-size=100m --log-opt max-file=5 --iptables=false --experimental=true --storage-driver overlay2
   
   $ ps axu | grep dockerd
     root      9381  0.5  1.5 333692 59836 pts/0    Sl+  14:14   0:00 dockerd --data-root=/var/lib/docker --log-opt max-size=100m --log-opt max-file=5 --iptables=false --experimental=true --storage-driver overlay2
   
   $ docker info
   Containers: 0
    Running: 0
    Paused: 0
    Stopped: 0
   Images: 0
   Server Version: 18.03.1-ce
   Storage Driver: overlay2
   ...
   ```
   
   
   
## ④ 测试

# 三. Q&A
1. `systemctl start docker` 启动docker报以下错
   ```
   报错: Job for docker.service failed because the control process exited with error code. See "systemctl status docker.service" and "journalctl -xe" for details.
   
   解决方案: 命令默认 /usr/lib/systemd/system/docker.service 这个配置文件，我们需要做以下修改
   
   把 ExecStart=/usr/bin/dockerd -H fd:// --containerd=/run/containerd/containerd.sock 替换成  ExecStart=/usr/bin/dockerd -H fd:// --containerd=/run/containerd/containerd.sock
   ```
   
2. 


