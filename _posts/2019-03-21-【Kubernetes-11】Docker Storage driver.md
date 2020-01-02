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
Docker支持几种不同的 [storage driver](https://docs.docker.com/storage/storagedriver/select-storage-driver/)，storage driver控制镜像和容器如何存储在宿主机文件系统上，最常见的有以下几种。源码可以参考 [docker daemon storage driver](https://github.com/moby/moby/tree/master/daemon/graphdriver)

1. `overlay2`: 推荐首选，支持所有的Linux发行版本，不需要额外的配置，但对内核版本有些要求，支持的后端文件系统为 `xfs、ext4`
2. `aufs`: 不能使用 overlay2 情况下，Ubuntu 和 Debian 推荐使用 aufs，支持的后端文件系统为 `xfs、ext4`
3. `devicemapper`: 不能使用 overlay2 情况下，CentOS 和 RHEL推荐使用 devicemapper，支持的后端文件系统为 `direct-lvm`

| Linux发行版本 | 推荐 storage driver | 候选 storage driver |
| :----- | :----- | :----- |
| Ubuntu | `overlay2` <br> 如果是 Ubuntu 14.04 运行在 内核 3.13 则推荐 `aufs` | overlay、devicemapper |
| Debian | `overlay2` <br> 老版本推荐 `aufs` 或 `devicemapper` | overlay |
| CentOS | `overlay2` | overlay、devicemapper |
| Fedora | `overlay2` | overlay、devicemapper |

`overlay 和 devicemapper 社区已经标记为deprecated，推荐使用 overlay2`

我们可以使用 `docker info` 查看当前docker daemon 配置的storage driver

```
$ docker info

Containers: 0
Images: 0
Storage Driver: overlay2
 Backing Filesystem: xfs
...
```

`特别注意: 如果我们修改了docker daemon的 storage driver，那么当前存在的镜像和容器都会变得不可访问，因为老的层没有办法被新的 storage driver 兼容使用。`

## ① aufs
[aufs](https://docs.docker.com/storage/storagedriver/aufs-driver/) 指的是联合文件系统，内核低于4.0 Ubuntu 和 Debian操作系统上推荐使用 `aufs` storage driver，如果内核版本高于 4.0 则推荐使用 `overlay2`，因为 aufs 比 overlay2 性能差一些。

联合文件系统的意思指的是把多个目录联合挂载到一个目录下，这个目录可以看到所有层的数据。如下图所示，镜像层和容器层联合挂载到 /var/lib/docker/aufs/mnt 某个子目录下。

![](https://github.com/chenguolin/chenguolin.github.io/blob/master/data/image/docker-storage-driver-aufs.jpg?raw=true)

aufs 默认把镜像和容器读写层存储到宿主机 /var/lib/docker/aufs/ 目录下，主要是以下3个子目录，镜像和容器在这3个子目录存储的内容还不太一样。
可以参考源码 [aufs storage driver](https://github.com/moby/moby/tree/master/daemon/graphdriver/aufs)

1. `diff/`: 存储每一层具体内容，包括镜像的每一层，以及容器的可读写层，当运行一个容器的时候会在该子目录下创建2个子目录，表示容器 `init层` 和 `读写层`
2. `layers/`: 存储每一层的meta信息，记录当前这层是基于哪些层构建而来
3. ·`mnt/`: 如果是镜像层则都是空目录，如果是容器读写层则表示联合挂载的目录

我们可以验证一下，先在公有云申请一个虚拟机，使用Ubuntu操作系统，然后安装好 docker 并使用 aufs 做为 storage driver

1. 查看docker基础信息，确认使用 aufs 做为storage driver，并且当前没有任何镜像和容器
```
$ docker info
Containers: 0
 Running: 0
 Paused: 0
 Stopped: 0
Images: 0
Server Version: 18.09.7
Storage Driver: aufs                   //使用aufs
 Root Dir: /var/lib/docker/aufs
 Backing Filesystem: extfs
 Dirs: 4
 Dirperm1 Supported: true
Logging Driver: json-file
Cgroup Driver: cgroupfs
```

2. 下载 ubuntu:18.04 镜像
```
$ docker pull ubuntu:18.04             //镜像总共4层
18.04: Pulling from library/ubuntu
2746a4a261c9: Pull complete
4c1d20cdee96: Pull complete
0d3160e1d0de: Pull complete
c8e37668deea: Pull complete
Digest: sha256:250cc6f3f3ffc5cdaa9d8f4946ac79821aafb4d3afc93928f0de9336eba21aa4
Status: Downloaded newer image for ubuntu:18.04
```

3. 查看 /var/lib/docker/aufs/ 目录内容
```
$ ls /var/lib/docker/aufs/
diff  layers  mnt

## 查看 diff 目录
$ ls /var/lib/docker/aufs/diff    (共4个子目录，子目录和层ID不是一一对应的)
45f79978fefa2597c8df8c2aa4a987ce9b5949c3a7814528d35edeaf3c90f587  6cad8d280e1cbb27586db468bdeb2ecdaa89b3940a3bbd0264da22828fa90b5e
62f2e3793e1671456d994c9638fceb8c7715c2625145185e41df5a6be5488cdd  ec9a24ea2d4f0f729e2952f13c0afb5c6792b6cf843278666ea26a51c3673b60

$ ls /var/lib/docker/aufs/diff/45f79978fefa2597c8df8c2aa4a987ce9b5949c3a7814528d35edeaf3c90f587
var

$ ls /var/lib/docker/aufs/diff/6cad8d280e1cbb27586db468bdeb2ecdaa89b3940a3bbd0264da22828fa90b5e
etc  sbin  usr	var

$ ls /var/lib/docker/aufs/diff/62f2e3793e1671456d994c9638fceb8c7715c2625145185e41df5a6be5488cdd
bin  boot  dev	etc  home  lib	lib64  media  mnt  opt	proc  root  run  sbin  srv  sys  tmp  usr  var

$ ls /var/lib/docker/aufs/diff/ec9a24ea2d4f0f729e2952f13c0afb5c6792b6cf843278666ea26a51c3673b60
run

## 查看 layers 目录
$ ls /var/lib/docker/aufs/layers  (共4个文件，文件名和层ID不是一一对应的)
45f79978fefa2597c8df8c2aa4a987ce9b5949c3a7814528d35edeaf3c90f587  6cad8d280e1cbb27586db468bdeb2ecdaa89b3940a3bbd0264da22828fa90b5e
62f2e3793e1671456d994c9638fceb8c7715c2625145185e41df5a6be5488cdd  ec9a24ea2d4f0f729e2952f13c0afb5c6792b6cf843278666ea26a51c3673b60

$ cat /var/lib/docker/aufs/layers/45f79978fefa2597c8df8c2aa4a987ce9b5949c3a7814528d35edeaf3c90f587
62f2e3793e1671456d994c9638fceb8c7715c2625145185e41df5a6be5488cdd

$ cat /var/lib/docker/aufs/layers/6cad8d280e1cbb27586db468bdeb2ecdaa89b3940a3bbd0264da22828fa90b5e
45f79978fefa2597c8df8c2aa4a987ce9b5949c3a7814528d35edeaf3c90f587
62f2e3793e1671456d994c9638fceb8c7715c2625145185e41df5a6be5488cdd

$ cat /var/lib/docker/aufs/layers/62f2e3793e1671456d994c9638fceb8c7715c2625145185e41df5a6be5488cdd  (第一层)

$ cat /var/lib/docker/aufs/layers/ec9a24ea2d4f0f729e2952f13c0afb5c6792b6cf843278666ea26a51c3673b60
6cad8d280e1cbb27586db468bdeb2ecdaa89b3940a3bbd0264da22828fa90b5e
45f79978fefa2597c8df8c2aa4a987ce9b5949c3a7814528d35edeaf3c90f587
62f2e3793e1671456d994c9638fceb8c7715c2625145185e41df5a6be5488cdd

## 查看 mnt 目录
$ ls /var/lib/docker/aufs/mnt  (共4个子目录，子目录和层ID不是一一对应的)
45f79978fefa2597c8df8c2aa4a987ce9b5949c3a7814528d35edeaf3c90f587  6cad8d280e1cbb27586db468bdeb2ecdaa89b3940a3bbd0264da22828fa90b5e
62f2e3793e1671456d994c9638fceb8c7715c2625145185e41df5a6be5488cdd  ec9a24ea2d4f0f729e2952f13c0afb5c6792b6cf843278666ea26a51c3673b60

$ ls /var/lib/docker/aufs/mnt/45f79978fefa2597c8df8c2aa4a987ce9b5949c3a7814528d35edeaf3c90f587
$ ls /var/lib/docker/aufs/mnt/6cad8d280e1cbb27586db468bdeb2ecdaa89b3940a3bbd0264da22828fa90b5e
$ ls /var/lib/docker/aufs/mnt/62f2e3793e1671456d994c9638fceb8c7715c2625145185e41df5a6be5488cdd
$ ls /var/lib/docker/aufs/mnt/ec9a24ea2d4f0f729e2952f13c0afb5c6792b6cf843278666ea26a51c3673b60
```

4. 使用 ubuntu:18.04 启动一个容器
```
$ docker run -it -d ubuntu:18.04
2563a860d2a8cd85473eff64ef1b0e8270bd388d6f4c95eadce4524602c55494
```

5. 查看 /var/lib/docker/aufs/ 目录内容
```
$ ls /var/lib/docker/aufs/       //没变，还是3个子目录
diff  layers  mnt

## 查看 diff 目录
$ ls /var/lib/docker/aufs/diff    (共6个子目录，有2个是新增的)
45f79978fefa2597c8df8c2aa4a987ce9b5949c3a7814528d35edeaf3c90f587  6cad8d280e1cbb27586db468bdeb2ecdaa89b3940a3bbd0264da22828fa90b5e
62f2e3793e1671456d994c9638fceb8c7715c2625145185e41df5a6be5488cdd  ec9a24ea2d4f0f729e2952f13c0afb5c6792b6cf843278666ea26a51c3673b60
fc40c28dc98a1b467f27d6196414eac9623666c344c615b7cb480fe630c9e277             //新增的子目录，容器读写层
fc40c28dc98a1b467f27d6196414eac9623666c344c615b7cb480fe630c9e277-init        //新增的子目录，容器init层

$ ls /var/lib/docker/aufs/diff/fc40c28dc98a1b467f27d6196414eac9623666c344c615b7cb480fe630c9e277

$ ls /var/lib/docker/aufs/diff/fc40c28dc98a1b467f27d6196414eac9623666c344c615b7cb480fe630c9e277-init
dev  etc

## 查看 layers 目录
$ ls /var/lib/docker/aufs/layers  (共6个文件，有2个是新增的)
45f79978fefa2597c8df8c2aa4a987ce9b5949c3a7814528d35edeaf3c90f587  6cad8d280e1cbb27586db468bdeb2ecdaa89b3940a3bbd0264da22828fa90b5e
62f2e3793e1671456d994c9638fceb8c7715c2625145185e41df5a6be5488cdd  ec9a24ea2d4f0f729e2952f13c0afb5c6792b6cf843278666ea26a51c3673b60
fc40c28dc98a1b467f27d6196414eac9623666c344c615b7cb480fe630c9e277             //新增的子目录，容器读写层
fc40c28dc98a1b467f27d6196414eac9623666c344c615b7cb480fe630c9e277-init        //新增的子目录，容器init层

$ cat /var/lib/docker/aufs/layers/fc40c28dc98a1b467f27d6196414eac9623666c344c615b7cb480fe630c9e277  
fc40c28dc98a1b467f27d6196414eac9623666c344c615b7cb480fe630c9e277-init
ec9a24ea2d4f0f729e2952f13c0afb5c6792b6cf843278666ea26a51c3673b60
6cad8d280e1cbb27586db468bdeb2ecdaa89b3940a3bbd0264da22828fa90b5e
45f79978fefa2597c8df8c2aa4a987ce9b5949c3a7814528d35edeaf3c90f587
62f2e3793e1671456d994c9638fceb8c7715c2625145185e41df5a6be5488cdd

$ cat /var/lib/docker/aufs/layers/fc40c28dc98a1b467f27d6196414eac9623666c344c615b7cb480fe630c9e277-init
fc40c28dc98a1b467f27d6196414eac9623666c344c615b7cb480fe630c9e277-init
ec9a24ea2d4f0f729e2952f13c0afb5c6792b6cf843278666ea26a51c3673b60
6cad8d280e1cbb27586db468bdeb2ecdaa89b3940a3bbd0264da22828fa90b5e
45f79978fefa2597c8df8c2aa4a987ce9b5949c3a7814528d35edeaf3c90f587
62f2e3793e1671456d994c9638fceb8c7715c2625145185e41df5a6be5488cdd

## 查看 mnt 目录
$ ls /var/lib/docker/aufs/mnt  (共6个子目录，有2个是新增的)
45f79978fefa2597c8df8c2aa4a987ce9b5949c3a7814528d35edeaf3c90f587  6cad8d280e1cbb27586db468bdeb2ecdaa89b3940a3bbd0264da22828fa90b5e
62f2e3793e1671456d994c9638fceb8c7715c2625145185e41df5a6be5488cdd  ec9a24ea2d4f0f729e2952f13c0afb5c6792b6cf843278666ea26a51c3673b60
fc40c28dc98a1b467f27d6196414eac9623666c344c615b7cb480fe630c9e277             //新增的子目录，容器读写层
fc40c28dc98a1b467f27d6196414eac9623666c344c615b7cb480fe630c9e277-init        //新增的子目录，容器init层

$ ls /var/lib/docker/aufs/mnt/fc40c28dc98a1b467f27d6196414eac9623666c344c615b7cb480fe630c9e277
bin  boot  dev	etc  home  lib	lib64  media  mnt  opt	proc  root  run  sbin  srv  sys  tmp  usr  var

$ ls /var/lib/docker/aufs/mnt/fc40c28dc98a1b467f27d6196414eac9623666c344c615b7cb480fe630c9e277-init
```

了解了 aufs 的实现原理之后，我们看下使用 aufs 做为 storage driver 是如何控制容器是读写文件的。

1. 读文件 (考虑以下3种场景)
    + `文件不在容器读写层`: 从镜像上层到下层搜索要读的文件，找到之后直接读取
    + `文件只存在容器读写层`: 直接从读写层读取
    + `文件存在容器读写层和镜像层`: 直接从读写层读取，镜像层对应的文件会被屏蔽
2. 写文件 (考虑以下3种场景)
    + `第一次写文件`: 如果文件存在镜像层，则从镜像层拷贝到读写层，并进行写入；如果文件不存在镜像层，则在读写层创建一个新文件并写入
    + `删除文件`: 直接在读写层创建一个 whiteout 文件，阻止容器访问被删除文件，看起来像删除文件。实际上是屏蔽掉镜像层文件，镜像层并没有改动（因为镜像层是只读）
    + `删除目录`: 直接在读写层创建一个 opaque 文件，阻止容器访问被删除目录，看起来像删除了目录。实际上是屏蔽掉镜像层目录，镜像层并没有改动（因为镜像层是只读）
    + `重命名目录`: aufs没有完成支持 [rename(2)](http://man7.org/linux/man-pages/man2/rename.2.html) 系统调用，业务需要校验是否发生错误

## ② devicemapper

## ③ overlay2








