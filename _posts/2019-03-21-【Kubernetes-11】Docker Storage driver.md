---
layout:  post  # 使用的布局（不需要改）
catalog: true  # 是否归档
author: 陈国林 # 作者
tags:          #标签
    - Kubernetes
---

# 一. 概述
前几篇文章我们提到了在Docker中持久化数据可以使用 [Docker数据挂载](https://chenguolin.github.io/2019/03/17/Kubernetes-8-Docker%E6%95%B0%E6%8D%AE%E6%8C%82%E8%BD%BD/) 的方式，我们知道容器启动之后会在镜像层、init层之上加上读写层，在读写层我们可以临时存储容器内变更的相关文件。容器读写层允许临时存储文件底层原理是通过 Storage drivers 实现的，但是当容器删除的时候这些数据会一起被删除，这篇文章会介绍 Docker Storage drivers 几种实现机制。

在了解  Storage drivers 之前，我们先来回顾一下 容器镜像分层的概念。容器镜像是分层的，每一层代表Dockerfile文件的一个指令。例如以下 Dockerfile 的内容，利用这个 Dockerfile 构建的镜像会有4层，文件每一行会构建出一层，每一层只存储和上一层的差异，最终镜像由这4层组成。

```
FROM ubuntu:15.04
COPY . /app
RUN make /app
CMD python /app/app.py
```

![](https://github.com/chenguolin/chenguolin.github.io/blob/master/data/image/docker-image-layer-1.jpg?raw=true)

当我们使用这个镜像运行一个容器的时候，会在最上层添加一个读写层也称为 `容器层`，容器运行期间所有的变更都会存储在读写层。当容器被删除的时候读写层会被删除，但是镜像并不会有任何的改变。`容器和镜像最大的不同在于，容器启动之后会基于镜像加上读写层`，因此如果使用相同的镜像启动多个容器，那么底层的镜像会被共享，极大提高了宿主机的磁盘空间利用率。(可以通过 `docker ps -s` 命令查看容器大概的磁盘空间占用)

![](https://github.com/chenguolin/chenguolin.github.io/blob/master/data/image/docker-container-layer-2.jpg?raw=true)

`Docker使用 Storage drivers 去管理镜像和容器的读写层的数据存储，不同 Storage driver 实现机制不同，但是分层和CoW是默认都会实现的。` Cow 指的是copy-on-write，表示当上一层想要访问下一层的某个文件的时候并不会立即copy到上一层，而是在真正要访问该文件的时候才进行copy，这样可以保证每一层都尽可能的小。

# 二. Storage drivers

## ① aufs

## ② devicemapper

## ③ overlay2








