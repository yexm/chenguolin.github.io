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
   $ dockerd --data-root=/var/lib/docker --log-opt max-size=100m --log-opt max-file=5 --iptables=false --metrics-addr=192.168.0.14:10270 --experimental=true --storage-driver overlay2 --dns 100.125.1.250 --dns 100.125.129.250 --dns 127.0.0.1 --dns 192.168.0.14 --dns-search default.svc.cluster.local --dns-search svc.cluster.local --dns-opt ndots:2 --dns-opt timeout:2 --dns-opt attempts:2
   
   $ ps axu | grep dockerd
     root      9381  0.5  1.5 333692 59836 pts/0    Sl+  14:14   0:00 dockerd --data-root=/var/lib/docker --log-opt max-size=100m --log-opt max-file=5 --iptables=false --metrics-addr=192.168.0.14:10270 --experimental=true --storage-driver overlay2 --dns 100.125.1.250 --dns 100.125.129.250 --dns 127.0.0.1 --dns 192.168.0.14 --dns-search default.svc.cluster.local --dns-search svc.cluster.local --dns-opt ndots:2 --dns-opt timeout:2 --dns-opt attempts:2
   
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

## ③ 配置worker
1. 安装Docker
   ```
   $ yum install -y https://download.docker.com/linux/centos/7/x86_64/stable/Packages/docker-ce-19.03.5-3.el7.x86_64.rpm
   
   ```

## ④ 测试
