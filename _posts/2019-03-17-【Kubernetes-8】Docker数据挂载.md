---
layout:  post  # 使用的布局（不需要改）
catalog: true  # 是否归档
author: 陈国林 # 作者
tags:          #标签
    - Kubernetes
---

# 一. 概述
默认情况下在容器内产生的所有文件都存储在容器的读写层，这就意味着有以下几个问题

1. 当容器不存在的时候这些数据会随着消失，同时另外一个容器很难访问当前容器内这些数据
2. 容器的读写层和宿主机紧密耦合，迁移数据难度大
3. 容器读写层数据都是由 storage-driver 进行管理，提供一个联合文件系统和内核进行交互，与直接操作宿主机的目录相比性能会差一些

Docker 提供了 `Volumes` 和 `Bind Mounts` 和2种数据挂载的方案，当容器销毁的时候数据还能持久化存储，如果我们是在Linux上跑docker容器还可以使用 `tmpfs Mounts`。

![](https://github.com/chenguolin/chenguolin.github.io/blob/master/data/image/docker-volume-1.png?raw=true)

1. `Volumes`: 存储在宿主机指定目录 `/var/lib/docker/volumes/`，非容器进程无法操作当前目录  (`Docker数据挂载最佳方式`)
2. `Bind Mounts`: 可以存储在宿主机任何一个目录，非容器进程可以操作它们
3. `tmpfs Mounts`: 只能存储在宿主机内存里，没有办法写到文件系统

# 二. Volumes (数据卷)
`Volumes` 是Docker数据挂载的首选方式，`Volumes` 由Docker管理和创建，可以通过 `docker volume create` 命令显式的创建，当产生一个Volume的时候它存储在宿主机的某一个目录上，当Volume挂载到容器的时候这个目录会被挂载到容器。一个Volume允许被多个容器同时挂载，当容器不再运行的时候，Volume不会被删除除非我们通过 `docker volume prune` 命令删除。[docker volumes](https://docs.docker.com/storage/volumes/)

Docker Volume可以提供很多有用的特性  
1. Volume可以在容器之间共享和重用，容器间传递数据将变得高效方便
2. Volume内数据的修改会立马生效，无论是容器内操作还是本地操作
3. Volume的更新不会影响镜像，解藕了应用和数据
4. Volume会一直存在，直到没有容器使用，可以安全地卸载它

![](https://github.com/chenguolin/chenguolin.github.io/blob/master/data/image/docker-volume-2.png?raw=true)

使用举例
```
1. 创建Volume
   docker volume create my-vol
2. 列出Volume
   docker volume ls
3. 查看Volume
   docker volume inspect my-vol
4. 删除Volume
   docker volume rm my-vol
5. 启动容器并挂载Volume (读写)
   docker run -d --name devtest --mount source=myvol2,target=/app nginx:latest
   或
   docker run -d --name devtest -v myvol2:/app nginx:latest
6. 查看容器信息
   docker inspect devtest
   "Mounts": [
       {
           "Type": "volume",
           "Name": "myvol2",
           "Source": "/var/lib/docker/volumes/nginx-vol/_data",
           "Destination": "/app",
           "Driver": "local",
           "Mode": "",
           "RW": true,
           "Propagation": ""
       }
    ],
7. 启动容器并挂载Volume (只读)
   docker run -d --name devtest --mount source=myvol2,target=/app,readonly nginx:latest
   或
   docker run -d --name devtest -v myvol2:/app:ro nginx:latest
8. 查看容器信息
   docker inspect devtest
   "Mounts": [
       {
           "Type": "volume",
           "Name": "myvol2",
           "Source": "/var/lib/docker/volumes/nginx-vol/_data",
           "Destination": "/app",
           "Driver": "local",
           "Mode": "",
           "RW": false,
           "Propagation": ""
       }
    ],
```

# 三. bind Mounts (绑定挂载)
`Bind Mounts`是Docker最早支持的方式，我们可以将宿主机的任何文件或目录（绝对路径）挂载到容器中。

1. 挂载的文件或目录可以被任何进程修改，因此有时候容器中修改了该文件或目录将会影响其他进程。
2. 如果挂载的文件或目录在宿主机上不存在将会自动创建，如果要挂载的文件或目录在容器内也存在，那么容器内和宿主机同名的文件或目录会被隐藏。
3. 使用该方式不能通过 docker volume 管理，推荐使用 volume 方式。

![](https://github.com/chenguolin/chenguolin.github.io/blob/master/data/image/docker-volume-3.png?raw=true)

`docker run`命令运行容器的时候，使用`-v`标记可以在容器内创建一个数据卷，多次使用`-v`标记可以创建多个数据卷。

使用 bind mount 启动容器
```
# 只读方式：--mount type=bind,source="$(pwd)"/target,target=/app,readonly
$ docker run -d \
  -it \
  --name devtest \
  --mount type=bind,source="$(pwd)"/target,target=/app \
  nginx:latest

# 只读方式：-v "$(pwd)"/target:/app:ro
$ docker run -d \
  -it \
  --name devtest \
  -v "$(pwd)"/target:/app \
  nginx:latest

使用 -v 和 --volume 绑定主机不存在的文件或目录，将会自动创建。始终创建的是一个目录。
使用 --mount 绑定主机上不存在的文件或目录，则不会自动创建，会产生一个错误。

$ docker inspect devtest  //查看容器信息
"Mounts": [
    {
        "Type": "bind",
        "Source": "/tmp/source/target",
        "Destination": "/app",
        "Mode": "",
        "RW": true,
        "Propagation": "rprivate"
    }
],
```

bind Mounts技术，允许你将一个目录或文件挂载到一个指定的目录下，这时候对挂载点进行的操作只是发生在被挂载的目录上，原挂载点的内容不受影响。例如把/home挂载到/test目录下，相当于把/test的inode重定向到/home目录下，对/test目录的任何操作实际修改的是/home目录。

![](https://static001.geekbang.org/resource/image/95/c6/95c957b3c2813bb70eb784b8d1daedc6.png)

# 四. tmpfs Mounts (tmpfs挂载)
`tmpfs Mounts`，仅存储在宿主机的内存中，不会写入宿主机的文件系统，使用它的情况一般是，对安全比较重视以及不需要持久化数据。tmpfs 挂载不能在容器间共享。tmpfs 只能在 Linux 容器上工作，不能在 windows 容器上工作。

容器使用tmpfs
```
$ docker run -d \
  -it \
  --name tmptest \
  --mount type=tmpfs,destination=/app \
  nginx:latest

$ docker run -d \
  -it \
  --name tmptest \
  --tmpfs /app \
  nginx:latest
  
$ docker container inspect tmptest  //查看容器信息
"Tmpfs": {
    "/app": ""
},
```

# 五. K8s数据挂载
K8s也提供了Volume配置，但是和Docker volume还不太一样。K8s Volume提供了在容器中挂载外部存储的能力，就是为了持久化你容器中的数据，能让它重建或者删除，数据依然存在。[kubernetes volumes](https://kubernetes.io/docs/concepts/storage/volumes/)

注意: `K8s hostpath和emptyDir类型在docker底层都是采用bind Mounts技术，可以通过docker inspect查看容器具体信息。`
