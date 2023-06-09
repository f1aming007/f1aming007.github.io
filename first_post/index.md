# Kubernets-Pod


# pod

### pod 的属性

- 凡是调度、网络、存储、以及安全相关的属性， 都是pod级别的

### **NodeSelector**

- 用户将Pod 与Node进行绑定的字段

```go
apiVersion: v1
kind: Pod
...
spec:
 nodeSelector:
   disktype: ssd
```

### **NodeName**

- 一旦pod 这个字段被赋值， Kubernetes项目就会被认为这个POd以及经过调度， 调度的结果就是复制的节点名字

### HostAliases

- 定义了Pod的Hosts文件**（比如 /etc/hosts）里的内容**，用法如下：

```go
apiVersion: v1
kind: Pod
...
spec:
  hostAliases:
  - ip: "10.1.2.3"
    hostnames:
    - "foo.remote"
    - "bar.remote"
...
```

pod启动之后、 /etc/hosts 文件内容如下

```go
cat /etc/hosts
# Kubernetes-managed hosts file.
127.0.0.1 localhost
...
10.244.135.10 hostaliases-pod
10.1.2.3 foo.remote
10.1.2.3 bar.remote
```

### shareProcessNamespace=true：

```go
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  shareProcessNamespace: true
  containers:
  - name: nginx
    image: nginx
  - name: shell
    image: busybox
    stdin: true
    tty: true
```

- pod 的容器共享 PID Namespace

### **ImagePullPolicy**

- 默认是Always ，每次都出重新拉取
- Never： 意味着Pod永远不会主动拉取这个镜像，
- IfNotPresent： 只在宿主机不存在这个镜像时才拉取

### lifecycle

- 在容器发生变化的时候触发一系列“钩子”

```go
apiVersion: v1
kind: Pod
metadata:
  name: lifecycle-demo
spec:
  containers:
  - name: lifecycle-demo-container
    image: nginx
    lifecycle:
      postStart:
        exec:
          command: ["/bin/sh", "-c", "echo Hello from the postStart handler > /usr/share/message"]
      preStop:
        exec:
          command: ["/usr/sbin/nginx","-s","quit"]
```

### pod对象在kubernetes中的生命周期

1. Pending：这个状态意味着， Pod的Yaml 文件以及提交给了kubernetes， API对象已经被创建并保存在Etcd 当中， 但是， 这个Pod 里有些容器因为某些原因不能被顺利创建， 比如， 调度不成功 
2. Running： Pod已经调度成功， 跟一个具体的节点绑定，它包含的容器已经被创建，并且至少有一个已经运行成功
3. Succeeded： Pod的所有容器都运行成功， 并且已经退出，
4. Failed： 这个状态下， Pod 里至少有一个容器以不正常的状态退出， 这个砖头的出现， 意味着你想办法Debug这个容器的应用， 
5. unknown： 异常状态， 意味着Pod的状态不能持续的被kubelet汇报给kube-apiserver， 很有可能是主从节点的通信出现了问题

### projected Volume 四种

1. Secret
2. ConfigMap
3. Downward API 
4. ServiceAccountToken

### secret

- 把Pod想要访问的加密数据存在Etcd 中， 可以通过Pod的容器挂载Volume的方式， 访问这些Secret 保存的信息

### configmap

- 保存的是不需要加密的， 应用所需的配置信息

### Downward API

- 让Pod里的容器能够直接获取到这个Pod  API对象本身的信息

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-downwardapi-volume
  labels:
    zone: us-est-coast
    cluster: test-cluster1
    rack: rack-22
spec:
  containers:
    - name: client-container
      image: k8s.gcr.io/busybox
      command: ["sh", "-c"]
      args:
      - while true; do
          if [[ -e /etc/podinfo/labels ]]; then
            echo -en '\n\n'; cat /etc/podinfo/labels; fi;
          sleep 5;
        done;
      volumeMounts:
        - name: podinfo
          mountPath: /etc/podinfo
          readOnly: false
  volumes:
    - name: podinfo
      projected:
        sources:
        - downwardAPI:
            items:
              - path: "labels"
                fieldRef:
                  fieldPath: metadata.labels
```

- 支持的字段
    - spec.nodeName - 宿主机名字
    - status.hostIP - 宿主机 IP
    - metadata.name - Pod 的名字
    - metadata.namespace - Pod 的 Namespace
    - status.podIP - Pod 的 IP

### Service Account

Service Account 对象的作用，就是 Kubernetes 系统内置的一种“服务账户”，它是 Kubernetes 进行权限分配的对象

Service Account 的授权信息和文件，实际上保存在它所绑定的一个特殊的 Secret 对象里的。这个特殊的 Secret 对象，就叫作**ServiceAccountToken**

**这种把 Kubernetes 客户端以容器的方式运行在集群里，然后使用 default Service Account 自动授权的方式，被称作“InClusterConfig”，也是我最推荐的进行 Kubernetes API 编程的授权方式。**

### 容器的健康检查和恢复机制

- 容器定义一个监控检查“探针”， kubelet就会根据这个Probe的返回值决定这个容器的状态，而不是直接以容器进行是否运行（来自 Docker 返回的信息）作为依据。这种机制，是生产环境中保证应用健康存活的重要手段。

```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    test: liveness
  name: test-liveness-exec
spec:
  containers:
  - name: liveness
    image: busybox
    args:
    - /bin/sh
    - -c
    - touch /tmp/healthy; sleep 30; rm -rf /tmp/healthy; sleep 600
    livenessProbe:
      exec:
        command:
        - cat
        - /tmp/healthy
      initialDelaySeconds: 5  ## 在容器启动后5s后开始执行
      periodSeconds: 5        ## 每5s执行一次
```

Kubernetes 里的**Pod 恢复机制**，也叫 restartPolicy

### 基本原理

1. 只有Pod 的 **restartPolicy 指定的策略允许重启的容器， 那么这个Pod就会保持Running状态，并进行容器重启**
2. 对于包含多个容器的Pod，**只有它里面所有的容器都进入异常状态后，Pod 才会进入 Failed 状态**
。在此之前，Pod 都是 Running 状态。此时，Pod 的 READY 字段会显示正常容器的个数

### PodPreset

PodPreset里定义的内容， 只会在Pod API对象被创建之前追加这个对象本身上， 而不会影响热河Pod的控制的定义

PodPreset 这样专门用来对 Pod 进行批量化、自动化修改的工具对象
