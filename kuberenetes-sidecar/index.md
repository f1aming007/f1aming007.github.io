# Kuberenetes Sidecar模式


#  Kubernetes 容器设计模式之边车模式
[sidecar](https://kubernetes.io/zh-cn/blog/2023/08/25/native-sidecar-containers/)
{{< figure src="/images/sidecar01.png" title="loki01" >}}
sidecar 边车模式 简单来说就是加装在摩托车旁边，用来拓展现有功能的能力，可以坐上更多的人或者物， 边车模式 像软件工程的代理模式，对服务进行包装， 使其不改变原来的功能，拓展原来的服务

`sidecar` 模式 是用来解决微服务的服务治理问题
<!--more-->
![](media/17025176918040.jpg)

## 介绍

`日志收集` 和 `服务健康检查` 这两个附加能力，来举例说明在不同时期 Sidecar 模式的实现方式

### Docker
{{< figure src="/images/sidecar02.png" title="loki01" >}}
![](media/17025184732286.jpg)

这种容器的设计模式是不正确的， 它将所有的`Sidecar` 的能力和主应用程序打包在了一起， 变成了一个富容器，容器应该是解决某个问题的功能单元，让容器保持单一用途才是正确的方式

#### 第一阶段
所有微服务治理的相关功能比如 `限流熔断`、`流量控制` 、 `服务限流` 等等，和应用程序紧耦合在一起， 业务逻辑混合了各式各样非业务功能的代码

#### 第二阶段
1. 一个是为了简化开发将这些非业务功能性代码做了剥离，将提供的能力代码拆分为独立的类库，有部分采用了开源的成熟框架，使其和业务代码做集成。
2. 另外一件事是重构，将部分服务统一往 Golang 上做迁移


### Kubernetes
{{< figure src="/images/sidecar03.png" title="loki01" >}}
![](media/17025191157533.jpg)

`kubernetes`的`pod`在容器的基础上，做了更高一层抽象，一个`pod`内可以有多个容器存在， 他们共享了同一个  `Network Namespace`， 并且可以声明共享一个`Volume`

左边是主应用程序所在的容器，右边两个是 `服务治理` 相关的辅助容器，也就是我们本篇文章的主题，可以称之为 `Sidecar` 容器

可以在 `pod` 中启动一个或多个辅助容器， 来完成一些独立于主容器之外的工作，这种模式的好处是显而易见的，每个容器都有它自己单一的用途，真正做到了职责分离，可以说是基本上解决了上述 容器时代架构 遗留下的大部分问题

## Sidecar Pattern
本质上 就是 主应用和附加能力的应用他妈拥有共同的生命周期

`Sidecar Pattern` 可以说是现代云计算非常重要的设计模式，通过指责分离与容器的隔离特性， 能够降低容器的复杂度，同时能扩展并增强已有容器的功能。

## 使用方式
{{< figure src="/images/sidecar04.png" title="loki04" >}}
![](media/17025203772845.jpg)

Pod 的 YAML 结构
```yaml
cat <<EOF | kubectl create -f -
apiVersion: v1
kind: Pod
metadata:
  name: nginx-example
spec:
  containers:
  - name: application-container
    image: nginx:1.15.2
    imagePullPolicy: Always
    ports:
    - containerPort: 80
    volumeMounts:
    - name: shared-files
      mountPath: /usr/share/nginx/html

  - name: sidecar-container
    image: lqshow/busybox-curl:1.28
    # 你可以想象成你的前端应用的静态文件全部打包在 /project/dist 目录下
    command: ['/bin/sh', '-c', "echo 'Hello, World!' > /project/dist/index.html && sleep 3600"]
    volumeMounts:
    - name: shared-files
      mountPath: /project/dist
    workingDir: /project/dist

  volumes:
  # 通过同一个卷来共享数据，用来共享 Sidecar 静态文件
  - name: shared-files
    emptyDir: {}
EOF
```

将脚本放在 Kubernetes 集群內执行，并通过 port-forward 来验证结果。
```bash
kubectl port-forward pod/nginx-example 3000:80
Forwarding from 127.0.0.1:3000 -> 80
Forwarding from [::1]:3000 -> 80
```

```bash
curl localhost:3000
Hello, World!
```

## Service Mesh
Sidecar Pattern 的使用越来越普遍，尤其是在 Service Mesh 领域， 它可以说是将 Sidecar Pattern 玩出了极致，且非常符合云原生的理念
Service Mesh 重新定义了微服务治理。
{{< figure src="/images/sidecar04.png" title="loki05" >}}
![](media/17025206481598.jpg)

