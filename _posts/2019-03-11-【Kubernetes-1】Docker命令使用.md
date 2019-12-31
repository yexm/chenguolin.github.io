---
layout:  post  # 使用的布局（不需要改）
catalog: true  # 是否归档
author: 陈国林 # 作者
tags:          #标签
    - Kubernetes
---

# 一. Docker
docker 提供了 cli 工具，源码可以参考 [github docker cli](https://github.com/docker/cli)，具体命令可以参考 [docker CLI](https://docs.docker.com/engine/reference/run/)

```
$ docker
Usage:	docker [OPTIONS] COMMAND

A self-sufficient runtime for containers

Options:
      --config string      Location of client config files (default "/Users/zj-db0972/.docker")
  -c, --context string     Name of the context to use to connect to the daemon (overrides DOCKER_HOST env var and default context set with "docker context use")
  -D, --debug              Enable debug mode
  -H, --host list          Daemon socket(s) to connect to
  -l, --log-level string   Set the logging level ("debug"|"info"|"warn"|"error"|"fatal") (default "info")
      --tls                Use TLS; implied by --tlsverify
      --tlscacert string   Trust certs signed only by this CA (default "/Users/zj-db0972/.docker/ca.pem")
      --tlscert string     Path to TLS certificate file (default "/Users/zj-db0972/.docker/cert.pem")
      --tlskey string      Path to TLS key file (default "/Users/zj-db0972/.docker/key.pem")
      --tlsverify          Use TLS and verify the remote
  -v, --version            Print version information and quit

Management Commands:
  builder     Manage builds
  checkpoint  Manage checkpoints
  config      Manage Docker configs
  container   Manage containers
  context     Manage contexts
  image       Manage images
  network     Manage networks
  node        Manage Swarm nodes
  plugin      Manage plugins
  secret      Manage Docker secrets
  service     Manage services
  stack       Manage Docker stacks
  swarm       Manage Swarm
  system      Manage Docker
  trust       Manage trust on Docker images
  volume      Manage volumes

Commands:
  attach      Attach local standard input, output, and error streams to a running container
  build       Build an image from a Dockerfile
  commit      Create a new image from a container's changes
  cp          Copy files/folders between a container and the local filesystem
  create      Create a new container
  deploy      Deploy a new stack or update an existing stack
  diff        Inspect changes to files or directories on a container's filesystem
  events      Get real time events from the server
  exec        Run a command in a running container
  export      Export a container's filesystem as a tar archive
  history     Show the history of an image
  images      List images
  import      Import the contents from a tarball to create a filesystem image
  info        Display system-wide information
  inspect     Return low-level information on Docker objects
  kill        Kill one or more running containers
  load        Load an image from a tar archive or STDIN
  login       Log in to a Docker registry
  logout      Log out from a Docker registry
  logs        Fetch the logs of a container
  pause       Pause all processes within one or more containers
  port        List port mappings or a specific mapping for the container
  ps          List containers
  pull        Pull an image or a repository from a registry
  push        Push an image or a repository to a registry
  rename      Rename a container
  restart     Restart one or more containers
  rm          Remove one or more containers
  rmi         Remove one or more images
  run         Run a command in a new container
  save        Save one or more images to a tar archive (streamed to STDOUT by default)
  search      Search the Docker Hub for images
  start       Start one or more stopped containers
  stats       Display a live stream of container(s) resource usage statistics
  stop        Stop one or more running containers
  tag         Create a tag TARGET_IMAGE that refers to SOURCE_IMAGE
  top         Display the running processes of a container
  unpause     Unpause all processes within one or more containers
  update      Update configuration of one or more containers
  version     Show the Docker version information
  wait        Block until one or more containers stop, then print their exit codes

Run 'docker COMMAND --help' for more information on a command.
```

# 二. 命令
## ① 镜像
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

## ② 容器
1. 创建一个容器 (create命令创建的容器状态是Created) [docker create](https://docs.docker.com/engine/reference/commandline/create/)
   ```
   $ docker create -it busybox
   5f7b95237023aed2939e94afd3c8d8f14239671ac4e3bba771bde09afbdf0820
   
   $ docker ps -a | grep 5f7b95237023
   5f7b95237023       busybox       "sh"       50 seconds ago         Created         focused_rhodes
   ```
2. 启动一个容器 (启动一个已经创建或停止的容器) [docker start](https://docs.docker.com/engine/reference/commandline/start/)
   ```
   $ docker start 5f7b95237023
   5f7b95237023
   
   $ docker ps -a | grep 5f7b95237023
   5f7b95237023        busybox         "sh"      2 minutes ago     Up 11 seconds      focused_rhodes
   ```
3. 创建并启动容器 (类似先create再start) [docker run](https://docs.docker.com/engine/reference/commandline/run/)
   ```
   $ docker run -it -d busybox 
   797b8c7173acc3c255a7486bfeed17e379ca9429f8128cc37ded726f42d4fbae
   
   备注:
   -i：让容器的标准输入保持打开
   -t：让Docker分配一个伪终端并绑定到容器的标准输入上
   -d: 让容器在后台运行
   
   $ docker ps -a | grep 797b8c7173ac
   797b8c7173ac        busybox        "sh"      31 seconds ago     Up 29 seconds      musing_haslett
   ```
4. 查看容器进程  [docker ps](https://docs.docker.com/engine/reference/commandline/ps/)
   ```
   $ docker ps -a
   797b8c7173ac        busybox        "sh"      31 seconds ago     Up 29 seconds      musing_haslett
   5f7b95237023        busybox        "sh"      2 minutes ago      Up 11 seconds      focused_rhodes
   ...
   ```
5. 停止容器 (首先向容器发送SIGTERM信号，等待超时时间（默认为10秒）后，再发送SIGKILL信号来终止容器)  [docker stop](https://docs.docker.com/engine/reference/commandline/stop/)
   ```
   $ docker stop 797b8c7173ac
   797b8c7173ac
   
   $ docker ps -a | grep 797b8c7173ac
   797b8c7173ac      busybox       "sh"      4 minutes ago       Exited (137) 26 seconds ago      musing_haslett
   ```
6. 重启容器  [docker restart](https://docs.docker.com/engine/reference/commandline/restart/)
   ```
   $ docker restart 797b8c7173ac
   797b8c7173ac
   
   $ docker ps -a
   797b8c7173ac      busybox      "sh"       31 seconds ago     Up 29 seconds      musing_haslett
   ```
7. 进入容器 (/bin/sh进程加入容器进程Namespace，这样在shell就能看到容器内部试图)  [docker exec](https://docs.docker.com/engine/reference/commandline/exec/)
   ```
   $ docker exec -it 797b8c7173ac /bin/sh
   / # 
   ```
8. 杀死正在运行的容器  [docker kill](https://docs.docker.com/engine/reference/commandline/kill/)
   ```
   $ docker kill --signal=SIGHUP 797b8c7173ac
   ```
9. 删除容器  [docker rm](https://docs.docker.com/engine/reference/commandline/rm/)
   ```
   $ docker rm -f 797b8c7173ac
   797b8c7173ac
   
   备注:
   -f，--force=false  强行终止并删除一个运行中的容器
   ```
10. 查看容器信息  [docker inspect](https://docs.docker.com/engine/reference/commandline/inspect/)
    ```
    $ docker inspect cea71c57fd06
    [
    {
        "Id": "cea71c57fd06afb567e71b1e237286d77af1d59663dc3e5a9ae0d5dc37843659",
        "Created": "2017-12-30T01:44:30.3634171Z",
        "Path": "sh",
        "Args": [],
        "State": {
    ......
    ```
11. 查看容器日志  [docker logs](https://docs.docker.com/engine/reference/commandline/logs/)
    ```
    $ docker logs -f --tail 100 e2df23840e1e
    ...
    ```

## ③ Daemon
1. 查看当前docker信息  [docker info](https://docs.docker.com/engine/reference/commandline/info/)
   ``` 
   $ docker info
   Client:
     Debug Mode: false

   Server:
     Containers: 136
       Running: 31
   ......
   ```
2. 查看docker版本  [docker version](https://docs.docker.com/engine/reference/commandline/version/)
   ```
   $ docker version
   Client: Docker Engine - Community
   Version:           19.03.4
   API version:       1.40
   Go version:        go1.12.10
   Git commit:        9013bf5
   Built:             Thu Oct 17 23:44:48 2019
   OS/Arch:           darwin/amd64
   Experimental:      false

   Server: Docker Engine - Community
   ```
3. 监听docker daemon进程事件流(Docker daemon会监听Unix域套接字：/var/run/docker.sock)  
   `curl --unix-socket /var/run/docker.sock http://localhost/events`
   
   
   
