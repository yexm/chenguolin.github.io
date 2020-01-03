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
Docker支持几种不同的 [storage driver](https://docs.docker.com/storage/storagedriver/select-storage-driver/)，storage driver控制镜像和容器如何存储在宿主机文件系统上，最常见的有以下3种，另外几种由于使用比较少就不再这里介绍。相关源码可以参考 [docker daemon storage driver](https://github.com/moby/moby/tree/master/daemon/graphdriver)

1. `overlay2`: 推荐首选，支持所有的Linux发行版本，不需要额外的配置，但对内核版本有些要求，支持的后端文件系统为 `xfs、ext4`
2. `aufs`: 不能使用 overlay2 情况下，Ubuntu 和 Debian 推荐使用 aufs，支持的后端文件系统为 `xfs、ext4`
3. `devicemapper`: 不能使用 overlay2 情况下，CentOS 和 RHEL推荐使用 devicemapper，支持的后端文件系统为 `direct-lvm`

| Linux发行版本 | 推荐 storage driver | 候选 storage driver |
| :----- | :----- | :----- |
| Ubuntu | `overlay2` <br> 如果是 Ubuntu 14.04 运行在 内核 3.13 则推荐 `aufs` | overlay、devicemapper |
| Debian | `overlay2` <br> 老版本推荐 `aufs` 或 `devicemapper` | overlay |
| CentOS | `overlay2` | overlay、devicemapper |
| Fedora | `overlay2` | overlay、devicemapper |

`备注: overlay 和 devicemapper 社区已经标记为deprecated，推荐使用 overlay2`

我们可以使用 `docker info` 查看当前docker daemon 配置的storage driver

```
$ docker info

Containers: 0
Images: 0
Storage Driver: overlay2
 Backing Filesystem: xfs
...
```

`特别注意: 如果我们修改了docker daemon的 storage driver，那么当前存在的镜像和容器都会变得不可访问，因为老的层没有办法被新的 storage driver 兼容使用。另外，不建议在容器的读写层临时存储大量的数据，一般来说如果有比较大的存储需要建议使用 Docker数据挂载 的方式。`

## ① aufs
[aufs](https://docs.docker.com/storage/storagedriver/aufs-driver/) 指的是联合文件系统，内核低于4.0 Ubuntu 和 Debian操作系统上推荐使用 `aufs` storage driver，如果内核版本高于 4.0 则推荐使用 `overlay2`，因为 aufs 比 overlay2 性能差一些。

联合文件系统的意思指的是把多个目录联合挂载到一个目录下，这个目录可以看到所有层的数据。如下图所示，镜像层和容器层联合挂载到 /var/lib/docker/aufs/mnt 某个子目录下。

![](https://github.com/chenguolin/chenguolin.github.io/blob/master/data/image/docker-storage-driver-aufs.jpg?raw=true)

aufs 默认把镜像和容器读写层存储到宿主机 /var/lib/docker/aufs/ 目录下，主要是以下3个子目录，镜像和容器在这3个子目录存储的内容还不太一样。
可以参考源码 [aufs storage driver](https://github.com/moby/moby/tree/master/daemon/graphdriver/aufs)

1. `diff/`: 存储每一层具体内容，包括镜像的每一层，以及容器的可读写层，当运行一个容器的时候会在该子目录下创建2个子目录，表示容器 `init层` 和 `读写层`
2. `layers/`: 存储每一层的meta信息，记录当前这层是基于哪些层构建而来
3. ·`mnt/`: 如果是镜像层则都是空目录，如果是容器读写层则表示联合挂载的目录，做为容器rootfs使用

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

   // 查看 diff 目录
   $ ls /var/lib/docker/aufs/diff    (共4个子目录，子目录和层ID不是一一对应的)
   45f79978fefa2597c8df8c2aa4a987ce9b5949c3a7814528d35edeaf3c90f587      
   6cad8d280e1cbb27586db468bdeb2ecdaa89b3940a3bbd0264da22828fa90b5e
   62f2e3793e1671456d994c9638fceb8c7715c2625145185e41df5a6be5488cdd 
   ec9a24ea2d4f0f729e2952f13c0afb5c6792b6cf843278666ea26a51c3673b60

   $ ls /var/lib/docker/aufs/diff/45f79978fefa2597c8df8c2aa4a987ce9b5949c3a7814528d35edeaf3c90f587
   var

   $ ls /var/lib/docker/aufs/diff/6cad8d280e1cbb27586db468bdeb2ecdaa89b3940a3bbd0264da22828fa90b5e
   etc  sbin  usr	var

   $ ls /var/lib/docker/aufs/diff/62f2e3793e1671456d994c9638fceb8c7715c2625145185e41df5a6be5488cdd
   bin  boot  dev	etc  home  lib	lib64  media  mnt  opt	proc  root  run  sbin  srv  sys  tmp  usr  var
  
   $ ls /var/lib/docker/aufs/diff/ec9a24ea2d4f0f729e2952f13c0afb5c6792b6cf843278666ea26a51c3673b60
   run

   // 查看 layers 目录
   $ ls /var/lib/docker/aufs/layers  (共4个文件，文件名和层ID不是一一对应的)
   45f79978fefa2597c8df8c2aa4a987ce9b5949c3a7814528d35edeaf3c90f587  
   6cad8d280e1cbb27586db468bdeb2ecdaa89b3940a3bbd0264da22828fa90b5e
   62f2e3793e1671456d994c9638fceb8c7715c2625145185e41df5a6be5488cdd  
   ec9a24ea2d4f0f729e2952f13c0afb5c6792b6cf843278666ea26a51c3673b60

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

   // 查看 mnt 目录
   $ ls /var/lib/docker/aufs/mnt  (共4个子目录，子目录和层ID不是一一对应的)
   45f79978fefa2597c8df8c2aa4a987ce9b5949c3a7814528d35edeaf3c90f587     
   6cad8d280e1cbb27586db468bdeb2ecdaa89b3940a3bbd0264da22828fa90b5e
   62f2e3793e1671456d994c9638fceb8c7715c2625145185e41df5a6be5488cdd  
   ec9a24ea2d4f0f729e2952f13c0afb5c6792b6cf843278666ea26a51c3673b60

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

   // 查看 diff 目录
   $ ls /var/lib/docker/aufs/diff    (共6个子目录，有2个是新增的)
   45f79978fefa2597c8df8c2aa4a987ce9b5949c3a7814528d35edeaf3c90f587  
   6cad8d280e1cbb27586db468bdeb2ecdaa89b3940a3bbd0264da22828fa90b5e
   62f2e3793e1671456d994c9638fceb8c7715c2625145185e41df5a6be5488cdd  
   ec9a24ea2d4f0f729e2952f13c0afb5c6792b6cf843278666ea26a51c3673b60
   fc40c28dc98a1b467f27d6196414eac9623666c344c615b7cb480fe630c9e277             //新增的子目录，容器读写层
   fc40c28dc98a1b467f27d6196414eac9623666c344c615b7cb480fe630c9e277-init        //新增的子目录，容器init层

   $ ls /var/lib/docker/aufs/diff/fc40c28dc98a1b467f27d6196414eac9623666c344c615b7cb480fe630c9e277

   $ ls /var/lib/docker/aufs/diff/fc40c28dc98a1b467f27d6196414eac9623666c344c615b7cb480fe630c9e277-init
   dev  etc

   // 查看 layers 目录
   $ ls /var/lib/docker/aufs/layers  (共6个文件，有2个是新增的)
   45f79978fefa2597c8df8c2aa4a987ce9b5949c3a7814528d35edeaf3c90f587   
   6cad8d280e1cbb27586db468bdeb2ecdaa89b3940a3bbd0264da22828fa90b5e
   62f2e3793e1671456d994c9638fceb8c7715c2625145185e41df5a6be5488cdd  
   ec9a24ea2d4f0f729e2952f13c0afb5c6792b6cf843278666ea26a51c3673b60
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

   // 查看 mnt 目录
   $ ls /var/lib/docker/aufs/mnt  (共6个子目录，有2个是新增的)
   45f79978fefa2597c8df8c2aa4a987ce9b5949c3a7814528d35edeaf3c90f587    
   6cad8d280e1cbb27586db468bdeb2ecdaa89b3940a3bbd0264da22828fa90b5e
   62f2e3793e1671456d994c9638fceb8c7715c2625145185e41df5a6be5488cdd  
   ec9a24ea2d4f0f729e2952f13c0afb5c6792b6cf843278666ea26a51c3673b60
   fc40c28dc98a1b467f27d6196414eac9623666c344c615b7cb480fe630c9e277             //新增的子目录，容器读写层
   fc40c28dc98a1b467f27d6196414eac9623666c344c615b7cb480fe630c9e277-init        //新增的子目录，容器init层

   $ ls /var/lib/docker/aufs/mnt/fc40c28dc98a1b467f27d6196414eac9623666c344c615b7cb480fe630c9e277   (容器rootfs)
   bin  boot  dev	etc  home  lib	lib64  media  mnt  opt	proc  root  run  sbin  srv  sys  tmp  usr  var

   $ ls /var/lib/docker/aufs/mnt/fc40c28dc98a1b467f27d6196414eac9623666c344c615b7cb480fe630c9e277-init
   ```

了解了 aufs 的实现原理之后，我们看下使用 aufs 做为 storage driver 是如何控制容器是读写文件的。

1. 读文件 (考虑以下3种场景)
    + `文件不在容器读写层`: 从镜像上层到下层搜索要读的文件，找到之后直接读取
    + `文件只存在容器读写层`: 直接从读写层读取
    + `文件存在容器读写层和镜像层`: 直接从读写层读取，镜像层对应的文件会被屏蔽
2. 写文件 (考虑以下4种场景)
    + `第一次写文件`: 如果文件存在镜像层，则从镜像层拷贝到读写层，并进行写入；如果文件不存在镜像层，则在读写层创建一个新文件并写入
    + `删除文件`: 直接在读写层创建一个 [whiteout](https://github.com/opencontainers/image-spec/blob/master/layer.md#whiteouts) 文件，阻止容器访问被删除文件，看起来像删除文件。实际上是屏蔽掉镜像层文件，镜像层并没有改动（因为镜像层是只读）
    + `删除目录`: 直接在读写层创建一个 [opaque-whiteout](https://github.com/opencontainers/image-spec/blob/master/layer.md#opaque-whiteout) 文件，阻止容器访问被删除目录，看起来像删除了目录。实际上是屏蔽掉镜像层目录，镜像层并没有改动（因为镜像层是只读）
    + `重命名目录`: aufs没有完全支持 [rename(2)](http://man7.org/linux/man-pages/man2/rename.2.html) 系统调用，业务需要校验是否发生错误

## ② devicemapper
Device Mapper 是Linux内核卷管理技术框架，[devicemapper](https://docs.docker.com/storage/storagedriver/device-mapper-driver/) 正是使用该框架的部分功能来管理宿主机镜像和容器的。devicemapper 使用块级别操作块设备，而不是文件级别。

devicemapper 默认把镜像和容器读写层存储到宿主机 /var/lib/docker/devicemapper/ 目录下，主要是以下3个子目录，可以参考源码 [devicemapper storage driver](https://github.com/moby/moby/tree/master/daemon/graphdriver/devmapper)。`注意: 使用 devicemapper 默认会限制每个每个device存储空间为10GB，也就是说容器读写层临时存储空间不能超过10GB，这点和其它 storage driver 会有些区别。`

1. `devicemapper`: 记录每个镜像和容器层的基础设备的数据，由Device Mapper实现
2. `metadata`: 镜像和容器每层的meta信息
3. `mnt`: 如果是镜像层则子目录为空，如果是容器读写层表示挂载点，做为容器rootfs使用

![](https://github.com/chenguolin/chenguolin.github.io/blob/master/data/image/docker-storage-driver-devicemapper.jpg?raw=true)

我们可以验证一下，先在公有云申请一个虚拟机，使用Centos操作系统，然后安装好 docker 并使用 devicemapper 做为 storage driver

1. 查看docker基础信息，确认使用 devicemapper 做为storage driver，并且当前没有任何镜像和容器
   ```
   $ docker info
   Containers: 0
    Running: 0
    Paused: 0
    Stopped: 0
   Images: 0
   Server Version: 17.05.0-ce
   Storage Driver: devicemapper
    Pool Name: docker-253:1-134591-pool
    Pool Blocksize: 65.54kB
    Base Device Size: 10.74GB
    Backing Filesystem: xfs
    Data file: /dev/loop0
    Metadata file: /dev/loop1
    Data Space Used: 11.73MB
    Data Space Total: 107.4GB
    ...
   ```

2. 下载 ubuntu:18.04 镜像
   ```
   $ docker pull ubuntu:18.04
   18.04: Pulling from library/ubuntu
   2746a4a261c9: Pull complete
   4c1d20cdee96: Pull complete
   0d3160e1d0de: Pull complete
   c8e37668deea: Pull complete
   Digest: sha256:250cc6f3f3ffc5cdaa9d8f4946ac79821aafb4d3afc93928f0de9336eba21aa4
   ```

3. 查看 /var/lib/docker/devicemapper/ 目录内容
   ```
   $ cd /var/lib/docker/devicemapper/
   $ ls
   devicemapper  metadata  mnt
   
   // 查看devicemapper目录
   $ ls devicemapper/
   data  metadata
   
   // 查看metadata目录
   $ ls metadata
   base             
   transaction-metadata
   deviceset-metadata
   15648f4feb33eb9eb2043e85bfe428601781b3c9822e4360190b91c1f7049aa3  
   c5dda0293b58cc885407f9b74916cfdbb258e2df812211dd52bfb66c98c420a7
   24e076ed9f38f0531899bee72e388b41793ee89f8d110038580f5526c4a49d3d                                                    
   eef5460abf44ffd0c9dc2a6555707d682f9da3cfd70c8da7d539a1510fc20d02
   
   $ cat metadata/base
   {"device_id":1,"size":10737418240,"transaction_id":1,"initialized":true,"deleted":false}
   
   $ cat metadata/transaction-metadata
   {"open_transaction_id":5,"device_hash":"c5dda0293b58cc885407f9b74916cfdbb258e2df812211dd52bfb66c98c420a7","device_id":5} 
   
   $ cat metadata/deviceset-metadata
   {"next_device_id":1,"BaseDeviceUUID":"1a08da9a-c394-476f-a0c5-09769024c1ba","BaseDeviceFilesystem":"xfs"}
   
   $ cat metadata/15648f4feb33eb9eb2043e85bfe428601781b3c9822e4360190b91c1f7049aa3
   {"device_id":2,"size":10737418240,"transaction_id":2,"initialized":false,"deleted":false}
   
   $ cat metadata/c5dda0293b58cc885407f9b74916cfdbb258e2df812211dd52bfb66c98c420a7
   {"device_id":5,"size":10737418240,"transaction_id":5,"initialized":false,"deleted":false}
   
   $ cat metadata/24e076ed9f38f0531899bee72e388b41793ee89f8d110038580f5526c4a49d3d
   {"device_id":3,"size":10737418240,"transaction_id":3,"initialized":false,"deleted":false}
   
   $ cat metadata/eef5460abf44ffd0c9dc2a6555707d682f9da3cfd70c8da7d539a1510fc20d02
   {"device_id":4,"size":10737418240,"transaction_id":4,"initialized":false,"deleted":false}
   
   // 查看mnt目录
   $ ls mnt/                  （共4个子目录，目录名和镜像层ID不是一一对应的)
   15648f4feb33eb9eb2043e85bfe428601781b3c9822e4360190b91c1f7049aa3     
   c5dda0293b58cc885407f9b74916cfdbb258e2df812211dd52bfb66c98c420a7
   24e076ed9f38f0531899bee72e388b41793ee89f8d110038580f5526c4a49d3d  
   eef5460abf44ffd0c9dc2a6555707d682f9da3cfd70c8da7d539a1510fc20d02
   $ ls mnt/15648f4feb33eb9eb2043e85bfe428601781b3c9822e4360190b91c1f7049aa3/   (镜像层，子目录为空)
   $ ls mnt/c5dda0293b58cc885407f9b74916cfdbb258e2df812211dd52bfb66c98c420a7/   (镜像层，子目录为空)
   $ ls mnt/24e076ed9f38f0531899bee72e388b41793ee89f8d110038580f5526c4a49d3d/   (镜像层，子目录为空)
   $ ls mnt/eef5460abf44ffd0c9dc2a6555707d682f9da3cfd70c8da7d539a1510fc20d02/   (镜像层，子目录为空)
   ```

4. 使用 ubuntu:18.04 启动一个容器
   ```
   $ docker run -it -d ubuntu:18.04
   a4385ee8c358683fcb2bdebb9a9fac6fe902b18f36daf22655e6552761547a45
   ```

5. 继续查看 /var/lib/docker/devicemapper/ 目录内容
   ```
   $ cd /var/lib/docker/devicemapper/
   $ ls
   devicemapper  metadata  mnt
   
   // 查看devicemapper目录
   $ ls devicemapper/
   data  metadata
   
   // 查看metadata目录       （多了2个文件，代表容器启动后的init层和读写层）
   $ ls metadata
   base             
   transaction-metadata
   deviceset-metadata
   15648f4feb33eb9eb2043e85bfe428601781b3c9822e4360190b91c1f7049aa3  
   c5dda0293b58cc885407f9b74916cfdbb258e2df812211dd52bfb66c98c420a7
   24e076ed9f38f0531899bee72e388b41793ee89f8d110038580f5526c4a49d3d                                                    
   eef5460abf44ffd0c9dc2a6555707d682f9da3cfd70c8da7d539a1510fc20d02
   ceae62f8d450d03e56a9c5adab60be9b3eedecdb95b0709d60c6f499c68cac11-init  //新增的文件，容器init层
   ceae62f8d450d03e56a9c5adab60be9b3eedecdb95b0709d60c6f499c68cac11       //新增的文件，容器读写层
   
   $ cat metadata/ceae62f8d450d03e56a9c5adab60be9b3eedecdb95b0709d60c6f499c68cac11-init
   {"device_id":6,"size":10737418240,"transaction_id":6,"initialized":false,"deleted":false}
   
   $ cat metadata/ceae62f8d450d03e56a9c5adab60be9b3eedecdb95b0709d60c6f499c68cac11
   {"device_id":7,"size":10737418240,"transaction_id":7,"initialized":false,"deleted":false}
   
   // 查看mnt目录
   $ ls mnt/                  （多了2个子目录，代表容器启动后的init层和读写层)
   15648f4feb33eb9eb2043e85bfe428601781b3c9822e4360190b91c1f7049aa3     
   c5dda0293b58cc885407f9b74916cfdbb258e2df812211dd52bfb66c98c420a7
   24e076ed9f38f0531899bee72e388b41793ee89f8d110038580f5526c4a49d3d  
   eef5460abf44ffd0c9dc2a6555707d682f9da3cfd70c8da7d539a1510fc20d02
   ceae62f8d450d03e56a9c5adab60be9b3eedecdb95b0709d60c6f499c68cac11-init
   ceae62f8d450d03e56a9c5adab60be9b3eedecdb95b0709d60c6f499c68cac11
   
   $ ls mnt/ceae62f8d450d03e56a9c5adab60be9b3eedecdb95b0709d60c6f499c68cac11-init/   (空目录)
   
   $ ls mnt/ceae62f8d450d03e56a9c5adab60be9b3eedecdb95b0709d60c6f499c68cac11   (id 为文件，rootfs为目录)
   id  rootfs
   
   $ cat mnt/ceae62f8d450d03e56a9c5adab60be9b3eedecdb95b0709d60c6f499c68cac11/id
   15648f4feb33eb9eb2043e85bfe428601781b3c9822e4360190b91c1f7049aa3
   
   $ ls mnt/ceae62f8d450d03e56a9c5adab60be9b3eedecdb95b0709d60c6f499c68cac11/rootfs/  (容器rootfs)
   bin  boot  dev  etc  home  lib  lib64  media  mnt  opt  proc  root  run  sbin  srv  sys  tmp  usr  var
   ```
   
了解了 devicemapper 的实现原理之后，我们看下使用 devicemapper 做为 storage driver 是如何控制容器是读写文件的。

1. 读文件 [device-mapper-driver reading-files](https://docs.docker.com/storage/storagedriver/device-mapper-driver/#reading-files)
2. 写文件 [device-mapper-driver writing-files](https://docs.docker.com/storage/storagedriver/device-mapper-driver/#writing-files)

## ③ overlay2
OverlayFS 是和 aufs 类似的联合文件系统，但是比 aufs 更快实现更简单，Docker 基于 OverlayFS 实现了 2种 storage driver，原生的 `overlay` 和 更稳定的 `overlay2`。推荐使用 overlay2，因为它在 inode 利用率上比 overlay 更好，但是要求 Linux 内核版本要在 4.0 以上。

overlay2 默认把镜像和容器读写层存储到宿主机 /var/lib/docker/overlay2/ 目录下，`子目录 l 内是一序列软链接，用来避免挂载时超出页面大小的限制`。 可以参考源码 [devicemapper storage driver](https://github.com/moby/moby/tree/master/daemon/graphdriver/overlay2)。

![](https://github.com/chenguolin/chenguolin.github.io/blob/master/data/image/docker-storage-driver-overlay2.jpg?raw=true)

我们可以验证一下，先在公有云申请一个虚拟机，使用Centos操作系统，然后安装好 docker 并使用 overlay2 做为 storage driver

1. 查看docker基础信息，确认使用 overlay2 做为storage driver，并且当前没有任何镜像和容器
   ```
   $ docker info
   Containers: 0
    Running: 0
    Paused: 0
    Stopped: 0
   Images: 0
   Server Version: 18.09.7
   Storage Driver: overlay2
    Backing Filesystem: extfs
    Supports d_type: true
    Native Overlay Diff: true
   Logging Driver: json-file
   ...
   ```

2. 下载 ubuntu:18.04 镜像
   ```
   $ docker pull ubuntu:18.04
   18.04: Pulling from library/ubuntu
   2746a4a261c9: Pull complete
   4c1d20cdee96: Pull complete
   0d3160e1d0de: Pull complete
   c8e37668deea: Pull complete
   Digest: sha256:250cc6f3f3ffc5cdaa9d8f4946ac79821aafb4d3afc93928f0de9336eba21aa4
   ```

3. 查看 /var/lib/docker/overlay2/ 目录内容
   ```
   $  ls -lsrt /var/lib/docker/overlay2   （总共5个子目录，有4个是镜像层子目录）
   total 20
   4 drwx------ 2 root root 4096 Jan  3 09:36 l
   4 drwx------ 3 root root 4096 Jan  3 09:36 047be5ac5412a89993002441f0923e41be8d16f35bcb6dd439d4b06bf1a8f874
   4 drwx------ 4 root root 4096 Jan  3 09:36 ada356abbf425aa2cabdec1d143428c7c74a1e496c34d420d5f6b1ebce49edf0
   4 drwx------ 4 root root 4096 Jan  3 09:36 622a0001387cbf29dda4bee820753663bc8cb2b494f908645d015782d62ec81a
   4 drwx------ 4 root root 4096 Jan  3 09:36 fd9af11a1ea6213093ebc6e7825169600619e2e103241e64fa3a2cca31f511bd
   
   $ ls -lsrt /var/lib/docker/overlay2/l
   total 16
   4 lrwxrwxrwx 1 root root 72 Jan  3 09:36 FCRJ5STD2UA4NPGVWMKHLO5XAI ->  ../047be5ac5412a89993002441f0923e41be8d16f35bcb6dd439d4b06bf1a8f874/diff
   4 lrwxrwxrwx 1 root root 72 Jan  3 09:36 TVWRIBSWMGT2IWS3XC2DZNJYKC -> ../ada356abbf425aa2cabdec1d143428c7c74a1e496c34d420d5f6b1ebce49edf0/diff
   4 lrwxrwxrwx 1 root root 72 Jan  3 09:36 PV7QAP3BQHWRQUFGO55NCUZI4T -> ../622a0001387cbf29dda4bee820753663bc8cb2b494f908645d015782d62ec81a/diff
   4 lrwxrwxrwx 1 root root 72 Jan  3 09:36 IH5WPOJD3VL2UYCG5PJSELFWFB -> ../fd9af11a1ea6213093ebc6e7825169600619e2e103241e64fa3a2cca31f511bd/diff

   $ ls -lsrt /var/lib/docker/overlay2/047be5ac5412a89993002441f0923e41be8d16f35bcb6dd439d4b06bf1a8f874
   total 8
   4 -rw-r--r--  1 root root   26 Jan  3 09:36 link
   4 drwxr-xr-x 21 root root 4096 Jan  3 09:36 diff     （当前这层的内容）
   
   $ ls -lsrt /var/lib/docker/overlay2/ada356abbf425aa2cabdec1d143428c7c74a1e496c34d420d5f6b1ebce49edf0
   total 16
   4 drwx------ 2 root root 4096 Jan  3 09:36 work      （OverlayFS内部使用目录）
   4 -rw-r--r-- 1 root root   28 Jan  3 09:36 lower     （记录它的下一层）
   4 -rw-r--r-- 1 root root   26 Jan  3 09:36 link
   4 drwxr-xr-x 3 root root 4096 Jan  3 09:36 diff      （当前这层的内容）
 
   $ ls -lsrt /var/lib/docker/overlay2/622a0001387cbf29dda4bee820753663bc8cb2b494f908645d015782d62ec81a
   total 16
   4 drwx------ 2 root root 4096 Jan  3 09:36 work      （OverlayFS内部使用目录）
   4 -rw-r--r-- 1 root root   57 Jan  3 09:36 lower     （记录它的下一层）
   4 -rw-r--r-- 1 root root   26 Jan  3 09:36 link
   4 drwxr-xr-x 6 root root 4096 Jan  3 09:36 diff      （当前这层的内容）

   $ ls -slrt /var/lib/docker/overlay2/fd9af11a1ea6213093ebc6e7825169600619e2e103241e64fa3a2cca31f511bd
   total 16
   $ drwx------ 2 root root 4096 Jan  3 09:36 work      （OverlayFS内部使用目录）
   4 -rw-r--r-- 1 root root   86 Jan  3 09:36 lower     （记录它的下一层）
   4 -rw-r--r-- 1 root root   26 Jan  3 09:36 link
   4 drwxr-xr-x 3 root root 4096 Jan  3 09:36 diff      （当前这层的内容）
   ```

4. 使用 ubuntu:18.04 启动一个容器
   ```
   $ docker run -it -d ubuntu:18.04
   f488754f6f7d6c0d441f58108a6721cb6b1eba0e5a586fc74f520e18f4ca52dd
   ```

5. 继续查看 /var/lib/docker/overlay2/ 目录内容
   ```
   $ ls -lsrt /var/lib/docker/overlay2   （新增2个子目录，表示容器init层和读写层）
   total 20
   4 drwx------ 2 root root 4096 Jan  3 09:36 l
   4 drwx------ 3 root root 4096 Jan  3 09:36 047be5ac5412a89993002441f0923e41be8d16f35bcb6dd439d4b06bf1a8f874
   4 drwx------ 4 root root 4096 Jan  3 09:36 ada356abbf425aa2cabdec1d143428c7c74a1e496c34d420d5f6b1ebce49edf0
   4 drwx------ 4 root root 4096 Jan  3 09:36 622a0001387cbf29dda4bee820753663bc8cb2b494f908645d015782d62ec81a
   4 drwx------ 4 root root 4096 Jan  3 09:36 fd9af11a1ea6213093ebc6e7825169600619e2e103241e64fa3a2cca31f511bd
   4 drwx------ 4 root root 4096 Jan  3 10:10 4d2283401faac9253bf6ecbc47bf1068e6d83ae62b1037e96b73e0b0a0442341-init
   4 drwx------ 5 root root 4096 Jan  3 10:10 4d2283401faac9253bf6ecbc47bf1068e6d83ae62b1037e96b73e0b0a0442341
   
   $ ls -lsrt /var/lib/docker/overlay2/4d2283401faac9253bf6ecbc47bf1068e6d83ae62b1037e96b73e0b0a0442341-init
   total 16
   4 drwx------ 3 root root 4096 Jan  3 10:10 work
   4 -rw-r--r-- 1 root root  115 Jan  3 10:10 lower
   4 -rw-r--r-- 1 root root   26 Jan  3 10:10 link
   4 drwxr-xr-x 4 root root 4096 Jan  3 10:10 diff

   $ ls -lsrt /var/lib/docker/overlay2/4d2283401faac9253bf6ecbc47bf1068e6d83ae62b1037e96b73e0b0a0442341
   total 20
   4 drwxr-xr-x 1 root root 4096 Jan  3 10:10 merged    (lowerdir 和 upperdir联合挂载点，做为容器rootfs)
   4 -rw-r--r-- 1 root root  144 Jan  3 10:10 lower
   4 -rw-r--r-- 1 root root   26 Jan  3 10:10 link
   4 drwxr-xr-x 2 root root 4096 Jan  3 10:10 diff
   4 drwx------ 3 root root 4096 Jan  3 10:10 work
   
   // 查看项目的子目录
   $ ls -lsrt /var/lib/docker/overlay2/4d2283401faac9253bf6ecbc47bf1068e6d83ae62b1037e96b73e0b0a0442341-init/diff/etc
   total 0
   0 -rwxr-xr-x 1 root root  0 Jan  3 10:10 resolv.conf
   0 lrwxrwxrwx 1 root root 12 Jan  3 10:10 mtab -> /proc/mounts
   0 -rwxr-xr-x 1 root root  0 Jan  3 10:10 hosts
   0 -rwxr-xr-x 1 root root  0 Jan  3 10:10 hostname

   $ ls -lsrt /var/lib/docker/overlay2/4d2283401faac9253bf6ecbc47bf1068e6d83ae62b1037e96b73e0b0a0442341-init/diff/dev
   total 8
   4 drwxr-xr-x 2 root root 4096 Jan  3 10:10 shm
   4 drwxr-xr-x 2 root root 4096 Jan  3 10:10 pts
   0 -rwxr-xr-x 1 root root    0 Jan  3 10:10 console
   
   $ ls -lsrt /var/lib/docker/overlay2/4d2283401faac9253bf6ecbc47bf1068e6d83ae62b1037e96b73e0b0a0442341/merged
   total 76
   4 drwxr-xr-x 8 root root 4096 May 23  2017 lib
   4 drwxr-xr-x 2 root root 4096 Apr 24  2018 sys
   4 drwxr-xr-x 2 root root 4096 Apr 24  2018 proc
   4 drwxr-xr-x 2 root root 4096 Apr 24  2018 home
   4 drwxr-xr-x 2 root root 4096 Apr 24  2018 boot
   4 drwxr-xr-x 1 root root 4096 Dec  2 20:43 usr
   4 drwxr-xr-x 2 root root 4096 Dec  2 20:43 srv
   4 drwxr-xr-x 2 root root 4096 Dec  2 20:43 opt
   4 drwxr-xr-x 2 root root 4096 Dec  2 20:43 mnt
   4 drwxr-xr-x 2 root root 4096 Dec  2 20:43 media
   4 drwxr-xr-x 2 root root 4096 Dec  2 20:43 lib64
   4 drwxr-xr-x 1 root root 4096 Dec  2 20:43 var
   4 drwx------ 2 root root 4096 Dec  2 20:43 root
   4 drwxr-xr-x 2 root root 4096 Dec  2 20:43 bin
   4 drwxrwxrwt 2 root root 4096 Dec  2 20:43 tmp
   4 drwxr-xr-x 1 root root 4096 Dec 19 12:21 sbin
   4 drwxr-xr-x 1 root root 4096 Dec 19 12:21 run
   4 drwxr-xr-x 1 root root 4096 Jan  3 10:10 etc
   4 drwxr-xr-x 1 root root 4096 Jan  3 10:10 dev

   $ ls -lsrt /var/lib/docker/overlay2/4d2283401faac9253bf6ecbc47bf1068e6d83ae62b1037e96b73e0b0a0442341/diff
   total 0
   
   $ ls -lsrt /var/lib/docker/overlay2/4d2283401faac9253bf6ecbc47bf1068e6d83ae62b1037e96b73e0b0a0442341/work
   total 4
   4 d--------- 2 root root 4096 Jan  3 10:10 work
   ```

了解了 overlay2 的实现原理之后，我们看下使用 overlay2 做为 storage driver 是如何控制容器是读写文件的。

1. 读文件 [reading-files](https://docs.docker.com/storage/storagedriver/overlayfs-driver/#reading-files) (考虑以下3种场景)
    + `文件不在容器读写层`: 从镜像上层到下层搜索要读的文件，找到之后直接读取
    + `文件只存在容器读写层`: 直接从读写层读取
    + `文件存在容器读写层和镜像层`: 直接从读写层读取，镜像层对应的文件会被屏蔽
2. 写文件 [modifying-files-or-directories](https://docs.docker.com/storage/storagedriver/overlayfs-driver/#modifying-files-or-directories) (考虑以下4种场景)
    + `第一次写文件`: 如果文件存在镜像层，则从镜像层拷贝到读写层，并进行写入；如果文件不存在镜像层，则在读写层创建一个新文件并写入
    + `删除文件`: 直接在读写层创建一个 whiteout 文件，阻止容器访问被删除文件，看起来像删除文件。实际上是屏蔽掉镜像层文件，镜像层并没有改动（因为镜像层是只读）
    + `删除目录`: 直接在读写层创建一个 opaque 文件，阻止容器访问被删除目录，看起来像删除了目录。实际上是屏蔽掉镜像层目录，镜像层并没有改动（因为镜像层是只读）
    + `重命名目录`: overlay2只支持源和目标目录都是在读写层，没有完全支持 [rename(2)](http://man7.org/linux/man-pages/man2/rename.2.html) 系统调用，业务需要校验是否发生错误

