Docker 快速入门、核心概念和常用指令

## 1、基本概念与操作

### 1.1、安装

> Linux 是 Docker 的原生支持平台，所以建议在 Linux 下安装。
> CentOS 下安装 Docker，需要 7 及以上的发行版，建议使用 overlay2 存储驱动程序。

```
# 卸载已有 docker
sudo yum remove docker \
                  docker-client \
                  docker-client-latest \
                  docker-common \
                  docker-latest \
                  docker-latest-logrotate \
                  docker-logrotate \
                  docker-engine

# 添加安装源
sudo yum-config-manager \
    --add-repo \
    https://download.docker.com/linux/centos/docker-ce.repo

# 安装最新版
sudo yum install docker-ce docker-ce-cli containerd.io

# 启动
sudo yum install docker-ce docker-ce-cli containerd.io
```

### 1.2、镜像

> 本质上是只读的文件和文件夹组合，包含了容器运行时所需要的所有基础文件和配置信息。
> 操作：
> 1、拉取镜像 docker pull
> 如：docker pull nginx
>
> 2、重命名镜像 docker tag
> 如：docker tag nginx:latest mynginx:latest
>
> 3、查看镜像 docker image ls 或 docker images
>
> 4、删除镜像 docker rmi
>
> 如：docker rmi mynginx
>
> 5、构建镜像 docker build 或 docker commit
> 如：docker commit nginx mynginx:lastest
> docker build 相对复杂，但使用较多

### 1.3、容器

> 容器是镜像的运行实体、一个镜像可以创建出多个容器、运行容器本质是在容器内部创建该文件系统的读写副本。
>
> 生命周期：
> created：初建状态
> running：运行状态
> stopped：停止状态
> paused：暂停状态
> deleted：删除状态
>
> 操作：
> 1、创建并启动容器
> 创建：docker create -it --name=mynginx mynginx
> 启动：docker start mynginx
> 创建并启动：docker run -it --name=mynginx mynginx
>
> 2、终止容器
> docker stop mynginx
>
> 3、进入容器
> docker attach mynginx
> docker exec -it mynginx sh （使用较多）
>
> 4、删除容器
> docker rm mynginx
> 删除运行中的容器：docker rm -f mynginx
>
> 5、导出容器
> docker export mynginx > mynginx.tar
>
> 6、导入容器
> docker import mynginx.tar mynginx:import

### 1.4、仓库

> 存储和分发 Docker 镜像；注册服务器是存放仓库的实际服务器，可包含很多个仓库，每个仓库可以包含多个镜像。
>
> 公共仓库 docker hub  https://hub.docker.com/
> 登录：docker login
> 推送镜像到仓库：docker push
>
> 使用 distribution 构建私有仓库
> https://github.com/distribution/distribution
>
> docker run -d -p 5000:5000 --name registry registry:2.7
> docker push localhost:5000/mynginx

### 1.5、卷

> 可以绕过默认的联合文件系统，直接以文件或目录的形式存在于宿主机上。它解决了数据持久化和容器间共享数据的问题。
> 操作：
> 1、创建：docker volume create volume-name
>
> 2、-v 指定被持久化的路径，Docker 会自动为我们创建卷，并且绑定到容器中
> docker run -d --name=nginx-volume -v /usr/share/nginx/html nginx
>
> 3、查看：docker volume ls
>
> 4、卷详细信息：docker volume inspect volume-name
>
> 5、--mount 参数指定卷的名称
> docker run -d --name=nginx --mount source=volume-name,target=/usr/share/nginx/html nginx
>
> 6、删除卷：docker volume rm volume-name
>
> 7、卷之间数据共享：
> docker run --mount source=lv,target=/tmp/log --name=v-producer -it test
> docker run -it --name consumer --volumes-from v-producer test
>
> 8、卷与主机之间数据共享：
> docker run -v /data:/usr/local/data -it test

### 1.6、重要组件

> 1、Docker
>
> - docker，是 Docker 客户端，发送请求
> - dockerd，服务端入口，负责接收请求、返回结果
> - docker-init，容器的 1 号进程，管理子容器
> - docker-proxy，主机的网络流量转发到容器
>
> 2、containerd
>
> - containerd，负责容器的生命周期管理，如容器启动、停止等…
> - containerd-shim，作为容器进程的父进程，解耦 containerd 和真正的容器进程
> - ctr，containerd 的客户端，开发与调试时向 containerd 发送请求
>
> 3、运行时
>
> - runc，通过系统接口，创建、销毁容器

### 1.7、容器监控

> docker stats 可查看主机上所有容器的 CPU、内存、网络 IO、磁盘 IO、PID 等资源的使用情况。
> cAdvisor 是谷歌开源的一款通用的容器监控解决方案。
> 安装参考：
>
> https://www.jianshu.com/p/91f9d9ec374f
>
> 查看监控：
> http://localhost:8080
> http://localhost:8080/containers/
> http://localhost:8080/docker/

### 1.8、安全问题

> - 自身安全漏洞
> - 镜像中存在安全问题
> - Linux 主机内核隔离不够

## 2、实现原理

### 2.1、Namespace

> Namespace 是 Linux 内核的一个特性，该特性可以实现在同一主机系统中，对进程 ID、主机名、用户、文件名、网络和进程间通信等资源的隔离。
>
> Docker 使用了六种：
> Mount Namespace，挂载点隔离
> PID Namespace，进程隔离
> UTS Namespace，主机名隔离
> IPC Namespace，进程间通信隔离
> User Namespace，用户和用户组隔离
> Net Namespace，网络设备、IP 地址和端口等隔离

### 2.2、Cgroups

> 限制进程或者进程组的资源，如 CPU、内存、磁盘 IO 等。
> cgroups 的功能：
>
> - 限制资源的使用量
> - 不同的组可以有 CPU 、磁盘 IO 等资源不同的使用优先级
> - 计算控制组的资源使用情况
> - 控制进程的挂起或恢复

### 2.3、联合文件系统

> Union File System，一种分层的轻量级文件系统，可以把多个目录内容联合挂载到同一目录下，从而形成一个单一的文件系统。
>
> Docker 中最常用的联合文件系统有三种：AUFS、Devicemapper 和 OverlayFS。
>
> - AUFS 最早、最成熟；
> - Devicemapper，Linux 内核提供的框架，是一种映射块设备的技术框架。核心概念有映射设备（mapped device）、目标设备（target device）、映射表（map table），包含 loop-lvm 模式、direct-lvm 模式(生产使用)；
> - overlay2，更新更稳定，对 Linux 内核和 Docker 版本要求都较高。

### 2.4、网络实现

> CNM (Container Network Model) 是 Docker 发布的容器网络标准。
> Libnetwork 是开源的，使用 Golang 编写，完全遵循 CNM 网络规范，是 CNM 的官方实现。
>
> Libnetwork 包含四种主要的网络模型：
>
> - null 空网络模式，不提供容器网络
> - bridge 桥接模式，容器与容器之间互通
> - host 主机网络模式，容器内与主机网络互通
> - container 网络模式，容器放在同一网络通过 localhost 访问

## 3、其他相关

### 3.1、容器编排

> Docker 三种常用的编排工具：Docker Compose、Docker Swarm 和 Kubernetes。
>
> - Docker Compose 是 Docker 收购得来，本质是一个 python 脚本，可以在单个结点上管理和编排多个容器。
> - Docker Swarm 是 Docker 官方推出的容器集群管理工具，原生支持 Docker API，它的操作简单、支持 TLS 双向认证、使用 Raft 协议实现分布式。
> - Kubernetes，Google 借鉴内部 Borg 系统沉淀的技术设计实现，功能强大，目标是能够支撑数亿容器的运行；但其架构较为复杂，上手门槛高。

### 3.2、在 devops 中的作用

> DevOps 的整体目标是促进开发和运维人员之间的配合，并且通过自动化的手段缩短软件的整个交付周期，提高软件的可靠性。
>
> 通过 Docker 快速安装开发环境、Dockerfile 构建镜像快速集成、拉取镜像运行容器即可完成部署，结合容器编排工具可实现蓝绿发布。
>
> 助力了 DevOps 的发展。
>
> 可以快速持续集成与交付。

好了，以上就是今天给大家分享的内容。

