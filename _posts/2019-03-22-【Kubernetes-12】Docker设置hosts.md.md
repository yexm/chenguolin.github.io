---
layout:  post  # 使用的布局（不需要改）
catalog: true  # 是否归档
author: 陈国林 # 作者
tags:          #标签
    - Kubernetes
---

# 一. 背景
很多时候我们会需要测试某个域名是否能够正常访问，如果域名解析还未生效我们一般需要先通过设置 hosts 的方式来保证能够正常访问。但是在Docker容器环境下，设置hosts和传统的物理机会有些区别。

正常情况下我们会想到在Dockerfile里面设置/etc/hosts，Dockerfile内容如下所示。使用Dockerfile完成镜像构建之后，运行容器之后发现容器内 /etc/hosts 文件并没有 10.10.0.14 cgl.test.com 这行配置。
```
FROM alpine:3.5

RUN echo "10.10.0.14 cgl.test.com" >> /etc/hosts
```

问题的根本原因是 hosts 文件其实并不是存储在Docker镜像中的，`/etc/hosts, /etc/resolv.conf 和 /etc/hostname，是存在宿主机 /var/lib/docker/containers/{container-id}/ 这个目录，容器启动时是通过 mount 将这些文件挂载到容器内部的`。因此如果在容器中修改这些文件，修改部分不会存在于容器的可读写层，而是直接写入这3个文件中。Docker每次创建新容器时，会根据当前 docker0 下的所有节点的IP信息重新建立hosts文件。也就是说，你的修改会被Docker给自动覆盖掉。可以参考源码 [setupPathsAndSandboxOptions](https://github.com/moby/moby/blob/master/daemon/container_operations_unix.go#L376)

我们可以来验证一下上面提到的点，docker daemon 使用 aufs 做为storage driver


1. 运行一个容器
   ```
   $ docker run -it -d ubuntu:18.04
   6a702140194255457943a8976564289b5036a838e9abe10b6518d6f2d1dcebb2
   ```
   
2. 查看容器内的 /etc/hosts 等文件内容
   ```
   $ docker exec -it 6a7021401942 /bin/sh
   
   $ cat /etc/hosts
   127.0.0.1	localhost 
   ::1	localhost ip6-localhost ip6-loopback
   fe00::0	ip6-localnet
   ff00::0	ip6-mcastprefix
   ff02::1	ip6-allnodes
   ff02::2	ip6-allrouters
   172.18.0.3	6a7021401942

   $ cat /etc/resolv.conf
   # This file is managed by man:systemd-resolved(8). Do not edit.
   #
   # This is a dynamic resolv.conf file for connecting local clients directly to
   # all known uplink DNS servers. This file lists all configured search domains.
   #
   # Third party programs must not access this file directly, but only through the
   # symlink at /etc/resolv.conf. To manage man:resolv.conf(5) in a different way,
   # replace this symlink by a static file or a different symlink.
   #
   # See man:systemd-resolved.service(8) for details about the supported modes of
   # operation for /etc/resolv.conf.
   nameserver 183.60.83.19
   nameserver 183.60.82.98

   $ cat /etc/hostname
   6a7021401942
   ```
  
3. 查看宿主机 /var/lib/docker/containers/6a702140194255457943a8976564289b5036a838e9abe10b6518d6f2d1dcebb2/ 目录内容，发现 hosts、resolv.conf、hostname 文件和容器内一致
   ```
   $ ls /var/lib/docker/containers/6a702140194255457943a8976564289b5036a838e9abe10b6518d6f2d1dcebb2
   6a702140194255457943a8976564289b5036a838e9abe10b6518d6f2d1dcebb2-json.log  config.v2.json   hostname  mounts	      
   resolv.conf.hash              checkpoints								  hostconfig.json  hosts     resolv.conf
   
   $ cat /var/lib/docker/containers/6a702140194255457943a8976564289b5036a838e9abe10b6518d6f2d1dcebb2/hosts
   127.0.0.1	localhost
   ::1	localhost ip6-localhost ip6-loopback
   fe00::0	ip6-localnet
   ff00::0	ip6-mcastprefix
   ff02::1	ip6-allnodes
   ff02::2	ip6-allrouters
   172.18.0.3	6a7021401942
 
   $ cat /var/lib/docker/containers/6a702140194255457943a8976564289b5036a838e9abe10b6518d6f2d1dcebb2/resolv.conf
   # This file is managed by man:systemd-resolved(8). Do not edit.
   #
   # This is a dynamic resolv.conf file for connecting local clients directly to
   # all known uplink DNS servers. This file lists all configured search domains.
   #
   # Third party programs must not access this file directly, but only through the
   # symlink at /etc/resolv.conf. To manage man:resolv.conf(5) in a different way,
   # replace this symlink by a static file or a different symlink. 
   #
   # See man:systemd-resolved.service(8) for details about the supported modes of
   # operation for /etc/resolv.conf.
   nameserver 183.60.83.19
   nameserver 183.60.82.98
   
   $ cat /var/lib/docker/containers/6a702140194255457943a8976564289b5036a838e9abe10b6518d6f2d1dcebb2/hostname
   6a7021401942
   ```

4. 查看容器读写层对应 /etc/hosts 等文件内容，发现 hosts、resolv.conf、hostname 文件内容都是空的
   ```
   $ cat /var/lib/docker/aufs/mnt/4382a1de34d0aada2f0300955a8e720f97c667756b4b2c02e2f24a3f8e0e0e29/etc/hosts
   $ cat /var/lib/docker/aufs/mnt/4382a1de34d0aada2f0300955a8e720f97c667756b4b2c02e2f24a3f8e0e0e29/etc/resolv.conf
   $ cat /var/lib/docker/aufs/mnt/4382a1de34d0aada2f0300955a8e720f97c667756b4b2c02e2f24a3f8e0e0e29/etc/hostname
   ```
   
5. 我们可以尝试在容器内修改 /etc/hosts 文件，你会发现 /var/lib/docker/containers/{container-id}/hosts 这个文件跟着变了。于此同时，如果我们把当前容器删除并创建一个新的容器之后，会发现之前的变更会重置了。这些就不再这里验证了，验证起来也很简单。

6. 从上面的验证过程可以验证 /etc/hosts, /etc/resolv.conf 和 /etc/hostname 等文件确实是在容器启动的时候生成的，同时存储到 /var/lib/docker/containers/{container-id}/ 目录下。除此之外，容器启动的时候会把 /var/lib/docker/containers/{container-id}/ 目录下 hosts、resolv.conf、hostname 等文件挂载到容器内，这些文件不是存储在容器读写层。

因此，如果我们有需要改变容器 /etc/hosts 的需求，具体的解决方案详见下文。

# 二. 解决方案
## ① docker run 添加 --add-host 参数
我们在运行容器的时候添加 --add-host 参数，容器运行命令为 `docker run -it --add-host=cgl.test.com:10.10.0.14 cgl-test-hosts:v0.0.1`,查看 hosts 是否有设置成功，发现容器内 /etc/hosts 已经有对应的域名解析配置了。

```
/ # cat /etc/hosts
127.0.0.1	localhost
::1	localhost ip6-localhost ip6-loopback
fe00::0	ip6-localnet
ff00::0	ip6-mcastprefix
ff02::1	ip6-allnodes
ff02::2	ip6-allrouters
10.10.0.14	cgl.test.com        //设置成功了
172.17.0.2	243fa854d215
```

注意: 虽然docker build命令也提供了 --add-host 选项，但是其实并不会生效。具体的原因可以参考 `the --add-host feature during build is designed to allow overriding a host during build, but not to persist that configuration in the image.`

## ② 进入容器修改
直接进入容器修改是最简单的方式，缺点是修改完之后每次重新启动一个新的容器都需要重新改动，不是很方便。

```
$ docker run -it cgl-test-hosts:v0.0.1
/ # cat /etc/hosts
127.0.0.1	localhost
::1	localhost ip6-localhost ip6-loopback
fe00::0	ip6-localnet
ff00::0	ip6-mcastprefix
ff02::1	ip6-allnodes
ff02::2	ip6-allrouters
172.17.0.2	21ab34d0fe34

/ # echo "10.10.0.14 cgl.test.com" >> /etc/hosts         //写入容器内/etc/hosts文件

/ # cat /etc/hosts
127.0.0.1	localhost
::1	localhost ip6-localhost ip6-loopback
fe00::0	ip6-localnet
ff00::0	ip6-mcastprefix
ff02::1	ip6-allnodes
ff02::2	ip6-allrouters
172.17.0.2	21ab34d0fe34
10.10.0.14 cgl.test.com      //设置成功了
```

## ③ 通过脚本注入镜像
通过前面的介绍我们知道在Dockerfile内直接修改 /etc/hosts 文件是无效的，因此我们需要把配置放在一个shell脚本内，通过Dockerfile把脚本注入镜像，同时设置容器的启动的命令为该shell脚本。

Dockerfile 内容如下
```
FROM alpine:3.5

COPY run.sh /www/cgl/run.sh
RUN chmod 775 /www/cgl/run.sh

# run the script that starts container
ENTRYPOINT /www/cgl/run.sh
```

run.sh 脚本内容如下
```
#!/bin/sh

## set hosts
echo "10.10.0.14 cgl.test.com" >> /etc/hosts

## start application
## sleep 3600只是举例
sleep 3600
```

1. 镜像构建: docker build -t cgl-test-hosts:v0.0.2 .
2. 运行容器并查看是否设置成功

```
$ docker run -itd cgl-test-hosts:v0.0.2
  aadb29a5716dd97d3a042808dbe391b5337e6b87205f4b3db670c640776414cf
$ docker exec -it aadb29a5716dd97d3a042808dbe391b5337e6b87205f4b3db670c640776414cf /bin/sh
/ # cat /etc/hosts
127.0.0.1	localhost
::1	localhost ip6-localhost ip6-loopback
fe00::0	ip6-localnet
ff00::0	ip6-mcastprefix
ff02::1	ip6-allnodes
ff02::2	ip6-allrouters
172.17.0.2	aadb29a5716d
10.10.0.14 cgl.test.com       //设置成功了
```
