# Docker Command


# docker常用命令整理

## docker images

* docker image pull ：下载镜像
* docker image ls：列出本地存储的镜像,参数 --digests查看镜像的SHA26签名
* docker image inspect:展示镜像细节。包括镜像层数据和元数据。
* docker image rm：删除镜像。

* docker提供参数 --filter 来过滤docker image ls 命令返回镜像列表的内容
* 返回悬虚(dangling)镜像
```bash
lhf@lhf-virtual-machine:~$ docker image ls --filter dangling=true
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
<none>  <none>
```
* 没有标签的镜像称为悬虚镜像，列表显示<none>:<none>
* 使用docker image prune 移除全部悬虚镜像。加-a参数，Docker会额外移除没有被使用的镜像。

### docker支持的过滤方式
1. dangling： 可以指定true或false，返回悬虚镜像(true)和非悬虚镜像(false).
2. before: 需要镜像名称和ID作为参数，返回在之前被创建的镜像
3. since: before类似，返回需要指定镜像之后创建的全部镜像
4. label: 根据备注（label）的名称或者值
5. reference: 过滤标签lastest镜像

```bash
lhf@lhf-virtual-machine:~$ docker image ls --filter=reference="*:latest"
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
test                latest              e63fd667d16a        2 days ago          71.4MB
alpine              latest              965ea09ff2eb        4 days ago          5.55MB
ubuntu              latest              cf0f3ca922e0        7 days ago          64.2MB
centos              latest              0f3e07c0138f        3 weeks ago         220MB
```

* 通过--format 参数来通过go模板对输出内容格式化
* 只返回docker主机上镜像的大小属性
```bash
lhf@lhf-virtual-machine:~$ docker image ls --format "{{.Size}}"
71.4MB
5.55MB
64.2MB
220MB
```
* 只返回显示仓库、标签和大小的信息
```bash
lhf@lhf-virtual-machine:~$ docker image ls --format "{{.Repository}}:{{.Tag}}:{{.Size}}"
test:latest:71.4MB
alpine:latest:5.55MB
ubuntu:latest:64.2MB
centos:latest:220MB
```

### 通过CLI方式搜索Docker Hub
```bash
lhf@lhf-virtual-machine:~$ docker search alpine
NAME                                   DESCRIPTION                                     STARS               OFFICIAL            AUTOMATED
alpine                                 A minimal Docker image based on Alpine Linux…   5757                [OK]                
mhart/alpine-node                      Minimal Node.js built on Alpine Linux           444                                     
anapsix/alpine-java                    Oracle Java 8 (and 7) with GLIBC 2.28 over A…   427 
<snip>
```
```bash
lhf@lhf-virtual-machine:~$ docker search alpine --filter "is-official=true"
NAME                DESCRIPTION                                     STARS               OFFICIAL            AUTOMATED
alpine              A minimal Docker image based on Alpine Linux…   5757                [OK] 
```

* 使用参数 --digests 在本地查看镜像摘要
```bash
lhf@lhf-virtual-machine:~$ docker image ls --digests alpine
REPOSITORY          TAG                 DIGEST                                                                    IMAGE ID            CREATED             SIZE
alpine              latest              sha256:c19173c5ada610a5989151111163d28a67368362762534d8a8121ce95cf2bd5a   965ea09ff2eb        4 days ago          5.55MB
```

* 删除docker主机的全部镜像
```bash
lhf@lhf-virtual-machine:~$ docker image rm $(docker image ls -q) -f
```

## docker container
* docker container run:启动rongq
* Ctrl-PQ:断开Shell与容器的连接
* docker container ls:列出运行状态的容器
* docker container exec:允许用户在运行状态的容器，启动一个新的进程。
* docker container stop：停止运行中的容器。
* docker container start:重启处于停止状态的容器。
* docker container rm：删除停止状态的容器。
* docker container inspect:显示容器的配置细节和运行时信息。

* -it参数： 使当前重点连接到容器的shell终端上
```bash
$ docker container run -it ubuntu /bin/bash
```

* 快速清理容器
```bash
$ docker container  rm $(docker container ls -aq) -f
```

## docker应用容器化

* docker image build ：读取Dockerfile文件,将应用程序容器化
    * 使用-t参数文件镜像打标签
    * 使用-f参数指定任意路径下的Dockerfile
* Dockerfile中FROM指令，指定构建镜像的一个基础层
* Dockerfile中RUN指令，在镜像中执行命令，创建新的镜像层
* Dockerfile中COPY指令，将文件作为新的层添加到镜像中
* Dockerfile中EXPOSR指令，记录应用所使用的的网络端口
* Dockerfile中ENTRYPOINT指令，指定镜像已容器的方式启动后默认运行程序

* 查看镜像构建执行了那些指令
```bash
$ docker image history lhfdocker/web:latest
```

* 查看镜像的构建详情
```bash
$ docker image inspect lhfdocker/web:latest
```

## Docker Compose
* docker-compose up ：部署一个compose应用。默认读取docker-compose.yml文件，可以使有-f参数指定文件,-d 参数在后台启动
* docker-compose stop：停止compose应用的相关容器。可以通过docker-compose restart重新启动
* docker-compose rm：删除已停止的compose应用的容器，会删除容器和网络，不会删除卷和镜像。
* docker-compose restart 重启compose应用
* 如果compose应用进行了变更,需要重启才能生效
* docker-compose ps:列出compose应用的容器
    * 输出内容包括：状态、容器的运行命令，已经网络端口
* docker-compose down:停止并删除运行中compose 的应用。会删除容器和网络，不会删除卷和镜像

## docker Swarm
* docker swarm init 创建一个新的swarm,执行这个命令的节点称为管理节点
* docker swarm join-token 查询接入管理节点和工作节点到现有swarm时所使用的命令和Token
    * 增加管理节点——docker swarm join-token manager
    * 增加工作节点——docker swarm join-token work
* docker node ls 列出swarm所有节点及相关信息
* docker service create 创建新服务
* docker service ls 列出swarm中运行的服务
* docker service ps <service> 获取更多关于服务服务的信息
* docker service inspect 获取服务的详细信息
* docker service scale 用于对服务副本数量进行增减
* docker service update 对运行中的服务进行属性变更
* docker service logs 查看服务的日志
* docker service rm 从swarm删除服务

## docker network
* docker network ls ：列出运行在本地的docker主机的全部网络
* docker network create :创建新的docker网络。默认采用的是bridge 加-d参数指定(网络类型)
* docker network inspect:提供docker网络的详细配置信息。
* docker network prune:删除docker主机上全部未使用的网络。
* docker network rm :删除docker主机上指定的网络

## docker volume
* docker volume create ：创建新卷，默认使用的local驱动，加-d参数指定不同驱动
* docker volume ls :查看docker主机的全部卷
* docker volume inspect: 查看卷的详细信息
* docker volume prune ：删除未被容器和服务使用的卷
* docker volume rm:删除指定卷

## Docker Stack
* docker stack deploy： 用于根据stack文件部署和更新stack服务
* docker stack ls ：列出swarm集群中所有的stack
* docker stack ps: 列出某个已经部署的stack的相关信息。
* docker stack rm：从swarm集群中移除stack
