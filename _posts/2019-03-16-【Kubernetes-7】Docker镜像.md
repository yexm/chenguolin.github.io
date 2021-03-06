---
layout:  post  # 使用的布局（不需要改）
catalog: true  # 是否归档
author: 陈国林 # 作者
tags:          #标签
    - Kubernetes
---

# 一. 镜像介绍
Docker里面所有的容器都是基于镜像，Docker镜像是通过 Dockerfile 构建出来的，Docker镜像在设计上引入了`层`的概念，也就是说用户制作镜像的每个步骤都会生成一个层。当镜像构建完成后，因为`storage-driver`的不同镜像存储的目录也不一样，通常在`dockerd`启动时候会设定 `--storage-driver`。

最常见的 storage-driver 有以下3种
1. `aufs`: ubuntu使用aufs做为storage-driver
    * /var/lib/docker/aufs/diff/<id>: 存储镜像内容
    * /var/lib/docker/repositories-aufs: JSON文件包含本地镜像信息
2. `devicemapper`: devicemapper把镜像和容器存储在自己的虚拟设备
    * /var/lib/docker/devicemapper/metadata: 镜像meta信息
    * /var/lib/docker/devicemapper/mnt: 每个容器RootFs真实挂载目录
3. `overlay2`: OverlayFS是和AUFS相似的联合文件系统，被很多人认为是AUFS的继承
    * /var/lib/docker/overlay2: 镜像每一层都在这个目录下
    * /var/lib/docker/overlay2/***-init 这种目录是创建容器后新增的目录，实际上是在已有的镜像层上创建了一层容器层

我们可以知道Docker镜像本质是 `挂载在容器根目录上，用来作为容器进程提供隔离后执行环境的文件系统`，镜像是静态定义，容器是镜像运行时表现。Docker使用UnionFS（Union File System）将镜像多个层联合挂载到同一个目录下，这个目录会被挂载到容器根目录上，做为容器进程的根文件系统使用，即RootFS也就是镜像。

RootFS 由三个部分组成 `镜像层`、`init层`、`容器层`
1. 镜像层: 只读层也称为镜像层是 Rootfs 最下面的层，一般称为基础镜像。
2. init层: init层位于只读层和读写层中间，init层是Docker容器单独生成的的一个内部层，用于存放/etc/hosts、/etc/resolv.conf等信息，由Docker引擎在容器启动之前做好。
3. 容器层: 读写层也称为容器层位于 Rootfs 最上层，容器启动之后会在最上层加上读写层，读写层是专门用来存放修改 Rootfs 之后的增量，无论是增、删、改都保存在这一层，读写层保存的是容器镜像变化的部分，并不会对镜像本身做任何修改。在容器中读取、修改、删除文件的时候，会从上到下查找此文件，找到文件后就复制到容器层，然后做修改，修改之后就会覆盖掉下层的文件。`容器和镜像最大的差异是在容器层`

![](https://stackify.com/wp-content/uploads/2019/07/image-8-1024x929.png)

# 二. 常用命令
1. 构建镜像 (要求Dockerfile在当前目录下) [docker build](https://docs.docker.com/engine/reference/commandline/build/)
   ```
   $ docker build -t vieux/apache:v2.0 .
   Uploading context 10240 bytes
   Step 1/3 : FROM busybox
   Pulling repository busybox
   ......
   ---> b35f4035db3f
   Step 3/3 : CMD echo Hello world
   ---> Running in 02071fceb21b
   ---> f52f38b7823e
   Successfully built f52f38b7823e
   Removing intermediate container 9c9e81692ae9
   Removing intermediate container 02071fceb21b
   ``` 
2. 查看镜像   [docker images](https://docs.docker.com/engine/reference/commandline/images/)
   ```
   $ docker images
   REPOSITORY                TAG                 IMAGE ID            CREATED             SIZE
   <none>                    <none>              77af4d6b9913        19 hours ago        1.089 GB
   committ                   latest              b6fa739cedf5        19 hours ago        1.089 GB
   ......
   ```
3. 登录镜像仓库 [docker login](https://docs.docker.com/engine/reference/commandline/login/)
   ```
   $ docker login --username {name} --password {password} {registry url}
   Login Succeeded
   ```
4. 拉取镜像  [docker pull](https://docs.docker.com/engine/reference/commandline/pull/)
   ```
   $ docker pull ubuntu:14.04
   
   14.04: Pulling from library/ubuntu
   5a132a7e7af1: Pull complete
   fd2731e4c50c: Pull complete
   28a2f68d1120: Pull complete
   a3ed95caeb02: Pull complete
   Digest: sha256:45b23dee08af5e43a7fea6c4cf9c25ccf269ee113168c19722f87876677c5cb2
   Status: Downloaded newer image for ubuntu:14.04
   ```
5. 推送镜像 [docker push](https://docs.docker.com/engine/reference/commandline/push/)
   ```
   $ docker push ubuntu:14.04
   ...
   ```
6. 删除镜像 [docker rmi](https://docs.docker.com/engine/reference/commandline/rmi/)
   ```
   $ docker rmi ubuntu:14.04
   Untagged: ubuntu:14.04
   Untagged: ubuntu@sha256:fe301db49df08c384001ed752dff6d52b4305a73a7f608f21528048e8a08b51e
   ```
7. 镜像打标签 [docker tag](https://docs.docker.com/engine/reference/commandline/tag/)
   ```
   $ docker tag ubuntu:14.04 cgl-ubuntu:14.04
   ```

# 三. 镜像构建
1. 编写Dockerfile，如下为 [go-logcollector fluentd 镜像 Dockerfile](https://github.com/chenguolin/go-logcollector/blob/master/build/fluentd/Dockerfile)，有一个原则就是 `尽可能保证镜像精简，镜像只添加必须的内容`
   ```
   FROM alpine:3.6

   # RUN install packages
   RUN apk update && \
       apk add --no-cache ruby && \
       apk add --no-cache ruby-json && \
       apk add --no-cache ruby-irb && \
       apk add --no-cache ruby-dev && \
       apk add --no-cache lsof && \
       apk add --no-cache ca-certificates && \
       apk add --no-cache wget && \
       apk add --no-cache curl && \
       apk add --no-cache openssl && \
       apk add --no-cache build-base && \
       apk add --no-cache vim

   # RUN install fluent plugin
   RUN gem install fluentd -v 1.6.0 --no-ri --no-rdoc && \
       gem install fluent-plugin-kafka -v 0.6.6 --no-ri --no-rdoc && \
       gem install fluent-plugin-flowcounter -v 1.3.0 --no-ri --no-rdoc && \
       gem install fluent-plugin-prometheus -v 1.4.0 --no-ri --no-rdoc && \
       gem install fluent-plugin-copy_ex -v 0.1.0 --no-ri --no-rdoc && \
       gem install ruby-kafka -v 0.6.8 --no-ri --no-rdoc && \
       gem install oj -v 2.18.1 --no-ri --no-rdoc && \
       gem uninstall ruby-kafka -v 0.7.8 && \
       rm -rf /root/.gem && \
       mkdir -p /etc/fluent/conf.d

   # WORKDIR
   WORKDIR /www/log-collector/

   # COPY
   COPY fluent.conf /etc/fluent/
   COPY log-collector /www/log-collector/

   # ENTRYPOINT
   ENTRYPOINT ["./log-collector"]
   ```
2. `.dockerignore` 用于忽略一些文件（类似.gitignore），在构建镜像之前不会把这些文件上传给dockerd，加快构建速度
   ```
   .git
   .gitignore
   .idea
   .vscode
   Dockerfile*
   README.md
   LICENSE
   ```
3. 基础镜像选择上，我们一般会选择 [alpine](https://hub.docker.com/_/alpine/)，alpine体积小非常适合做为基础镜像
4. 镜像构建: `docker build .`

# 四. Dockerfile 最佳实践
1. 通过 `Dockerfile` 构建的镜像应当尽可能精简
2. 尽量不安装非必要的软件包
3. 一个容器只运行一个单独的实例，将具有耦合度的应用分别安装到不同的容器里面
4. 慎重引入新的数据层，最小化层数
5. 将准备安装的软件包按安装的字母顺序排列，这样可以避免重复安装软件包的情况，同时也有助于进行软件更新。通过添加 `\` 分割命令，增加代码的可读性
6. 尽量选择官方提供的镜像版本来作为基础镜像，减小镜像的体积，推荐使用 alpine
7. 将多条 `RUN` 命令使用`\`连接起来，这样更易于理解，方便维护。把复杂的或过长的 RUN 语句写成以 `\` 结尾的多行的形式，以提高可读性和可维护性
8. 为镜像定义一个比较通用的端口，比如提供 HTTP Web 服务的镜像，最好是暴露 80 端口
9. Dockerfile 的开头几行的指令应当固定下来，不要每次都随意更改，这样可以利用缓存
    + Dockerfile的每条指令都会将结果提交为新的镜像，下一条指令基于上一条指令的镜像进行构建。如果一个镜像拥有相同的父镜像和指令，Docker将会使用已有镜像而不是重新执行该指令
    + 因此，为了有效的利用缓存，尽量保持Dockerfile一致，并且尽量在末尾修改
    + 如不希望使用缓存，在执行 docker build 的时候加上参数 `--no-cache=true`
10. 通过 `–t` 标记来构建镜像，有助于用户管理每个创建的镜像，可读的标签可以帮助管理每个创建的镜像
11. 不要在 Dockerfile 中映射公有端口，在 Dockerfile 中你可以映射私有和公有端口，但永远不要通过 Dockerfile 映射公有端口。这样运行多个镜像的情况下会出现端口冲突的问题
    + EXPOSE 80:8080  # 80映射到host的8080 `(不提倡这种用法)`
    + EXPOSE 80 # 80会被docker随机映射一个端口
12. 使用 CMD 和 ENTRYPOINT 时，一定要用数组语法，而且 CMD 和 ENTRYPOINT 结合使用效果更好
13. 不要在构建中升级版本，不在容器中更新，更新交给基础镜像来处理
14. 在 Push 之前，现在本地构建运行一下，确保本地构建的镜像可以在任何地方正常运行
15. WORKDIR 的路径始终使用绝对路径可以保证指令的准确和可靠，使用 WORKDIR 来替代 RUN cd ... && do-something 这样难以维护的指令
16. 使用指令组合，比如 apt-get update 应该和 apt-get install 结合
17. 容器是短暂的，容器模型是进程而不是机器，不需要开机初始化。在需要时运行，不需要是停止，能够删除后重建，并且配置和启动的最小化
18. 最小化层数，需要掌握好 Dockerfile 的可读性和文件系统层数之间的平衡。控制文件系统层数的时候会降低 Dockerfile 的可读性。而 Dockerfile 可读性高的时候，往往会导致更多的文件系统层数
19. 虽然 ADD 与 COPY 功能类似，但推荐使用 COPY。COPY 只支持基本的文件拷贝功能，更加的可控。而 ADD 具有更多特定，比如tar文件自动提取，支持URL。通常需要提取 tarball 中的文件到容器的时候才会用到 ADD
20. FROM 命令应该包含基础镜像的完整仓库名和标签。Dockerfile中 FROM 应始终包含依赖的基础镜像的完整仓库名和标签，如使用 FROM debian:jessie 而不是 FROM debian


