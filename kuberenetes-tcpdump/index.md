# Kuberenetes  网络抓包  sniff


# 在Kubernetes下实现网络抓包（sniff）

如果 kubernetes 线上服务出现问题， 通过网络抓包，排查网络来定位问题
常见的抓包有几种方法
<!--more-->

## 在宿主机上抓包

首先定位pod内的容器被调度到哪个Node 上， 在相应的Node里， 对容器进行抓包

### 定位Pod 的containerID 以及它所运行的宿主机IP
在 Kubernetes 集群内执行下面这个指令，并从返回的结果中拿到两个信息

宿主机 IP = `172.18.0.xxx`
container ID = `78e91175699f.....`

```bash
# 注: 需要替换 namespace 和 pod name
➜ kubectl get pod \
  -n ${NAMESPACE} ${POD_NAME} \
  -o json|jq '.status|{hostIP: .hostIP, container: [.containerStatuses[]|{name: .name, containerID: .containerID}]}'
{
  "hostIP": "172.18.0.xxx",
  "container": [
    {
      "name": "linkerd-proxy",
      "containerID": "docker://78e91175699f8cc0a3b0ff87da97407c19c7a86706a5b74e2d86f4428a4de75a"
    },
    {
      "name": "nginx",
      "containerID": "docker://7a6f7eabc2d5112437d30ee8ec1aa7ef963e97c3d09c3bc63613a70d106d7d01"
    }
  ]
}
```

### 查询网络接口索引
通过ssh 登陆到pod所在的宿主机上， 然后在容器内执行 `cat /sys/class/net/eth0/iflink` ， 查找容器中的网卡与宿主机的veth 网卡直接的对应关系
```bash
# 注: 需要替换 container id
docker exec -it ${CONTAINER_ID} /bin/bash -c 'cat /sys/class/net/eth0/iflink'
10227
```

### 查找网络接口信息
在宿主机上做 ip link 操作
```bash
ip link |grep 10227
10227: calicf227ed888a@if4: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1440 qdisc noqueue state UP mode DEFAULT group default
```

### 在宿主机上抓包
通过tcpdump指令， 将报文保存到文件中
```bash
tcpdump -i calicf227ed888a@if4 -w /root/tcpdump.pcap 
```


## 在pod 内抓包

在目标pod 内抓包
```bash
# 注: -c ${CONTAINER_NAME} 是可选择的。如果 Pod 中仅包含一个容器，就可以忽略它
kubectl exec ${POD_NAME} -c ${CONTAINER_NAME} -- tcpdump -i eth0 -w - | wireshark -k -i -
```


## 通过 ephemeral containers 抓包
值得一提的是 Kubernetes 在 v1.16 [Alpha] 开始支持 Ephemeral Containers[2]，它正好可以解决上面提的 2 个痛点，临时容器对于排除交互式故障非常有用，Kubernetes 在 v1.23 [beta] 已经默认开启该功能了。

比如我使用 nicolaka/netshoot[3] 镜像用来调试，用法如下
```bash
# 注: 需要替换 pod name 和 container name
kubectl debug -i ${POD_NAME} \
  --image=nicolaka/netshoot \
  --target=${CONTAINER_NAME} -- tcpdump -i eth0 -w - | wireshark -k -i -
```


## 通过 Ksniff 抓包
ksniff 是一个 kubectl 的插件，它利用 tcpdump 和 Wireshark 对 Kubernetes 集群中的任何 Pod 启动远程抓包

ksniff 一般使用 krew 这个 kubectl 包管理器进行安装

### 安装 Krew
```bash
(
  set -x; cd "$(mktemp -d)" &&
  curl -fsSLO "https://github.com/kubernetes-sigs/krew/releases/download/v0.4.4/krew-linux_amd64.tar.gz" &&
  tar zxvf krew.tar.gz &&
  KREW=./krew-"$(uname | tr '[:upper:]' '[:lower:]')_$(uname -m | sed -e 's/x86_64/amd64/' -e 's/arm.*$/arm/')" &&
  "$KREW" install krew
)
export PATH="${KREW_ROOT:-$HOME/.krew}/bin:$PATH"
```

### 安装 sniff 插件

```bash
kubectl krew install sniff
```

### 执行 kubectl sniff 命令抓包
```bash
kubectl sniff ${POD_NAME} -n ${NAMESPACE}
```

执行 sniff 命令后，本地会自动启动 Wireshark 进行抓包，如下图
{{< figure src="/images/sniff02.png" title="sniff02" >}}

![](media/17028874913645.jpg)
以下是 sniff 运行的日志，我只提取了日志的关键部分

```bash
➜ kubectl sniff httpbin -n default

time="2022-02-20T19:56:13+08:00" level=info msg="using tcpdump path at: '/Users/linqiong/.krew/store/sniff/v1.6.1/static-tcpdump'"
time="2022-02-20T19:56:13+08:00" level=info msg="selected container: 'httpbin'"
```

从运行的日志来看，sniff 是将本地的 static-tcpdump 文件上传到 Pod 的指定容器的 /tmp 目录下，然后在容器内，通过运行以下命令来达到抓包的目的
```bash
/tmp/static-tcpdump -i any -U -w -
```

### 保存抓包文件
有时在生产环境我们可能无法直接在本地执行 kubectl，需要经过跳板机，这个时候我们可以将抓到的包保存成文件，然后再拷到本地使用 wireshark 分析。

只需加一个 -o 参数指定下保存的文件路径即可:

```bash
kubectl -n test sniff website-7d7d96cdbf-6v4p6 -o test.pcap
```
