# Kuberenetes Telepersence


# 用Telepresence 本地调试kubernetes中的微服务
github地址：[telepresence](https://github.com/telepresenceio/telepresence)
功能介绍：
{{< figure src="/images/tele01.png" title="Telepresence01" >}}

1. 快速的本地开发循环，无需等待容器构建/推送/部署
2. 能够使用他们最喜欢的本地工具（IDE、调试器等）
3. 能够运行无法在本地运行的大型应用程序

<!--more-->
![](media/17025228434831.jpg)

telepresence 主要是解决 Kubernetes  集群下开发微服务时遇到的日常问题， 
解决本地和集群内服务集成联调的问题。

## 微服务开发的痛点

一个平台如果微服务化后，至少也是几十个服务起步了吧，这还是在不包含基础设施服务的前提下。

接到一个需求，在某个服务下增加一个功能或者是要修复一个 bug，那么你接下来准备怎么做呢？

打算将几十个服务都在本机跑起来吗？且不说你不知道平台到底有多少个服务，光是想想配置这一整套环境都头大，再说你的机器是否有足够的资源能撑的起平台的运行，所以说 pure offline 的方式肯定不是一个可行的方向。


## 方法一： 利用端口转发
利用  Kubernetes  原生的特性  port-forward，将远端集群的两个服务，转发到本地，通过 localhost 的方式来访问，完成了服务的验证

Port forwarding k8s.Service.1
```bash
➜ kubectl port-forward svc/k8s-service-1 22222
Forwarding from 127.0.0.1:22222 -> 22222
Forwarding from [::1]:22222 -> 22222
```

Port forwarding k8s.Service.2
```bash
kubectl port-forward svc/k8s-service-2 22223
Forwarding from 127.0.0.1:22223 -> 22223
Forwarding from [::1]:22223 -> 22223
```
这个方法存在问题：
1. 开发者需要明确清楚依赖的目标服务
2. 开发者需要清楚目标服务在  Kubernetes 集群內的 Pod Name 或者 Service Name 以及端口号
3. 如果依赖的服务增多，每个服务都要手动或写脚本做端口转发，非常的复杂且不方便
4. 本地配置和部署配置不一致（本地使用 localhost，线上使用 svc）
5. 端口转发因为网络问题容易中断


## 方法二： 利用 kubefwd 工具
Kubefwd，它是一个用于端口转发  Kubernetes 中指定 namespace 下全部或部分服务的命令行工具。
使用  Kubefwd  后，本地环境也能使用  Kubernetes 集群內 svc + port 方式直接访问


> 一个端口转发的工具[kubewd](https://github.com/txn2/kubefwd)

{{< figure src="/images/tele02.png" title="Telepresence02" >}}
![](media/17025239857914.jpg)

```bash
Examples:
 kubefwd services --help
  kubefwd svc -n the-project
  kubefwd svc -n the-project -l env=dev,component=api
  kubefwd svc -n the-project -f metadata.name=service-name
  kubefwd svc -n default -l "app in (ws, api)"
  kubefwd svc -n default -n the-project
  kubefwd svc -n the-project -m 80:8080 -m 443:1443
  kubefwd svc -n the-project -z path/to/conf.yml
  kubefwd svc -n the-project -r svc.ns:127.3.3.1
  kubefwd svc --all-namespaces
```

Kubefwd将转发所有headlesss Service的Pod
```bash
sudo kubefwd svc -n the-project
```

转发namespace the-project下所有的带有label为system: wx的service：
```bash
sudo kubefwd svc -l system=wx -n the-project
```

**这种方法虽然方便，但是存在以下问题**

1. 跨多个 namespace 调试不方便
2. 单向调用（只能本地服务访问远程集群中的其他服务）


## 开发现状

### Local + Remote
利用 Kubernetes 原生的特性port-forward 命令或者 kubefwd 工具，将  Kubernetes 集群中的服务转发到本地，然后通过 local 的方式访问  Kubernetes 集群中的服务，这种方式可以解决开发过程中的一些联调问题。
1. 需要转发的服务配置不可控，且取决于团队成员是否都有 Kubernetes 基础，是否对平台的微服务架构都比较了解。
2. 不稳定，时不时需要重新做服务映射，特别是在远程用 VPN 连入公司内网情况下
3. 单向调用，只能从本地向集群发请求，集群内的流量请求无法劫持转发到本地

### Remote
利用内网自动化流水线的的方式，将代码发布到内网测试环境，即将服务持续的部署到内网  Kubernetes 测试环境中进行联调测试，以便提前发现问题。

1. 虽然该过程可以自动完成，但是每次调试至少需要经历以下步骤（Git commit -> docker build -> deploy to k8s cluster），耗时比较长，效率低下。
2. 一般每个团队都是共享内网开发测试环境的，这种 Remote Debug 方式会影响到团队内的其他成员的开发进度


## Telepresence
Telepresence 对基于 Kubernetes 的开发者来说是种非常强大的调试工具

1. Telepresence 是一个面向`Kubernetes`用户的开发测试环境治理的辅助工具，用于本地轻松开发和调试服务， 同时将服务代理到远程  Kubernetes 集群，无需等待容器做 build/push/deploy
2. 使用 Telepresence 开发者可以使用本地熟悉的 IDE 和调试工具运行一个服务，并提供对 Configmap、Secrets 和远程集群上运行的服务的完全访问，无缝与 Kubernetes 集群中的其他服务进行联调，让微服务本地开发不再难。
3. 可以在不用修改代码的情况下，让本地应用程序无感知接入到 `Kubernetes` 集群中，简单来说就是可直接使用集群内的 `PodIP`， `ClusterIP` 以及 `DNS` 域名来访问集群中的其他服务。
4. 因为 Telepresence 在 `kubernetes` 集群中运行的Pod 中部署了 双向网络代理，所以不再是单向代理，本地服务可以完全访问远程集群中的其他服务，同时远程集群中运行的服务也可以完全访问本地服务

### 安装方法
参考链接[telepresence](https://www.getambassador.io/docs/telepresence-oss/latest/quick-start?os=macos)
```bash
# Intel Macs

# 1. Download the latest binary (~105 MB):
sudo curl -fL https://app.getambassador.io/download/tel2oss/releases/download/v2.16.1/telepresence-darwin-amd64 -o /usr/local/bin/telepresence

# 2. Make the binary executable:
sudo chmod a+x /usr/local/bin/telepresence

# Apple silicon Macs

# 1. Download the latest binary (~101 MB):
sudo curl -fL https://app.getambassador.io/download/tel2oss/releases/download/v2.16.1/telepresence-darwin-arm64 -o /usr/local/bin/telepresence

# 2. Make the binary executable:
sudo chmod a+x /usr/local/bin/telepresence
```

### 本地环境
当执行完 `telepresence connect` ， 可以想象成你的本地环境就是 `Kubernetes` 集群中的一个 `pod`， 能够无感知接入到 `Kubernetes` 集群， Telepresence 让你的本地环境成为 `集群中的一部分` 

```bash
telepresence connect
```

开发者每次修改代码后，重新编译，推送镜像，部署到 Kubernetes 集群，然后调试，如果出现错误，再重复这样的步骤，这个是以往大部分开发者的日常行为。

大家也知道，下图中右侧虚框中的部分因为各种外部原因，常常是不可控的

{{< figure src="/images/tele03.png" title="Telepresence03" >}}
![](media/17025352313055.jpg)


Telepresence 是复杂的微服务架构原本只能从这种Remote 的验证方式得到解放
调整为开发和联调测试行为都发生在本地，这样对开发者的效率明显提升很多。

{{< figure src="/images/tele04.png" title="Telepresence04" >}}
![](media/17025358170320.jpg)

安装 Telepresence(macOS)
```bash
brew install datawire/blackbird/telepresence
```

## 案例
telepresence 默认会使用当前的 kubectl 的 current context 来进行请求。

一、在 Kubernetes 集群运行一个 hello-world 服务
```bash
kubectl run hello-world \
--image=datawire/hello-world \
--port=8000 \
--expose
```

二、 在本地启动 Telepresence，建立到集群的连接
```bash
# 将本地环境连接到远程 kubernetes 集群
➜ telepresence connect
Launching Telepresence Root Daemon
Launching Telepresence User Daemon
Connected to context development-private@xdp-bee (https://<clusterip>:6443)
```

三、查看连接状态
```bash
telepresence status
User Daemon: Running
  Version           : v2.17.1
  Executable        : /opt/homebrew/bin/telepresence
  Install ID        : 834d6f44-05a1-4b63-82ff-8de25208e3ce
  Status            : Connected
  Kubernetes server : https://192.168.0.65:6443
  Kubernetes context: k8s-65
  Connection name   : k8s-65-default
  Namespace         : default
  Manager namespace : ambassador
  Intercepts        : 0 total
Root Daemon: Running
  Version    : v2.17.1
  Version    : v2.17.1
  DNS        :
    Remote IP       : 127.0.0.1
    Exclude suffixes: [.com .io .net .org .ru]
    Include suffixes: []
    Timeout         : 8s
  Also Proxy : (0 subnets)
  Never Proxy: (1 subnets)
    - 192.168.0.65/32
Ambassador Cloud: Running
  Status      : Logged in
  User ID     : 8b474552-01e1-4c7c-8163-6c48141ffee1
  Account ID  : 6987f3d9-9f4e-4569-8045-1d563ef73f57
  User Name   : hongfeng liu
  Email       : fxqi1221@gmail.com
  Account Name: hongfeng
  Plan        : Free Trial
Traffic Manager: Connected
  Version : v2.17.1
Intercept Spec:
```

四、验证直接连接 hello-world 服务
```bash
curl http://hello-world.default:8000
Hello, world!
```

用浏览器访问该域名

{{< figure src="/images/tele05.png" title="Telepresence05" >}}
![](media/17025370475442.jpg)
Telepresence v2 通过 Kubernetes service 的 服务名称 + 命名空间 + 端口 访问服务

五、 退出守护进程
```bash
telepresence quit
Telepresence Root Daemon quitting... done
Telepresence User Daemon is already stopped
```


参考
https://mp.weixin.qq.com/s/583aqGHcjYkBLStWrxiknA

https://github.com/lqshow/telepresence-labs

https://imti.co/kubernetes-port-forwarding/

https://hackernoon.com/locally-developing-kubernetes-services-without-waiting-for-a-deploy-f63995de7b99

