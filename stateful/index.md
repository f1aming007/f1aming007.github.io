# Kubernets-StatefulSet


# StatefulSet

### 概念

StatefulSet 的设计其实非常容器理解， 它把真实世界里的应用状态， 抽象为两个情况

1. **拓扑状态：这种情况意味着， 应用的多个实例之间不是完全对等的关系， 这些应用实例， 不行按照某些顺序启动，比如应用的主节点A要先于节点B启动， 而如果你把A和B两个Pod删除调， 他们再次被创建出来也必须严格安装这个顺序才行， 并且， 新创建出来的Pod，必须和原理的Pod的网络标识一样， 这样原先的访问者才能使用同样的方法， 访问到这个新Pod。**
2. **存储状态：这种情况， 应用的多个实例分别绑定了不同的存储数据， 对于这些应用实例来说，Pod A第一次读取到的数据， 和隔了十分钟之后再次读取到的数据， 应该是同一份，哪怕再次期间Pod A被重新创建过，  这个情况 就是一个数据库应用的多个存储实例**

**StatefulSet的核心功能， 就是通过某种方式记录这些状态，然后再Pod被重新创建时， 能够为新的Pod恢复这些状态**

### Headless Service

1. **Service的VIP方式**：当我访问 10.0.23.1 这个 Service 的 IP 地址时，10.0.23.1 其实就是一个 VIP，它会把请求转发到该 Service 所代理的某一个 Pod 上
2. **Service的DNS方式：**这时候，只要我访问“my-svc.my-namespace.svc.cluster.local”这条 DNS 记录，就可以访问到名叫 my-svc 的 Service 所代理的某一个 Pod。
    1. **Normal Service**： 你访问“my-svc.my-namespace.svc.cluster.local”解析到的，正是 my-svc 这个 Service 的 VIP
    2. **Headless Service**： 你访问“my-svc.my-namespace.svc.cluster.local”解析到的，就是my-svc代理的某个Pod的IP地址，

区别在于Headless Service不需要分配一个VIP， 而是直接以DNS记录方式解析出被代理的Pod的IP地址

### Headless Service 对应的 YAML 文件：

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx
  labels:
    app: nginx
spec:
  ports:
  - port: 80
    name: web
  clusterIP: None
  selector:
    app: nginx
```

Headless Service

```yaml
<pod-name>.<svc-name>.<namespace>.svc.cluster.local
```

### StatefulSet 又是如何使用这个DNS记录来维持Pod的拓扑状态的？

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web
spec:
  serviceName: "nginx"
  replicas: 2
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.9.1
        ports:
        - containerPort: 80
          name: web
```

serviceName=nginx 字段，告诉 StatefulSet控制器， 在执行控制循环（Control Loop）的时候， 请使用nginx 这个 Headless Service 来保证 Pod 的“可解析身份”。

```bash
$ kubectl run -i --tty --image busybox dns-test --restart=Never --rm /bin/sh 
$ nslookup web-0.nginx
Server:    10.0.0.10
Address 1: 10.0.0.10 kube-dns.kube-system.svc.cluster.local
 
Name:      web-0.nginx
Address 1: 10.244.1.8
 
$ nslookup web-1.nginx
Server:    10.0.0.10
Address 1: 10.0.0.10 kube-dns.kube-system.svc.cluster.local
 
Name:      web-1.nginx
Address 1: 10.244.2.8
```

**Kubernetes 就是成功地将Pod的拓扑状态（比如：哪个节点先启动，哪个节点后启动），按照 Pod 的“名字 + 编号”的方式固定了下来**

### 总结

StatefulSet 这个控制器主要作用之一： 就是使用Pod模板创建Pod的时候，对他们进行编号， 并且按照编号顺序逐一完成创建工作， 而当StatefulSet 的”控制循环”发现Pod的“实际状态”与“期望状态”不一致， 需要新建或者删除Pod进行 “调谐”的时候，它会严格按照这些 Pod 编号的顺序，逐一完成这些操作。

# **深入理解StatefulSet（二）：存储状态**

### 工作原理

- 首信， **StatefulSet的控制器直接管理的是Pod**， ， 因为StatefulSet里的不同的Pod实例， 不想ReplicaSet中那样都是完全一样的， 而是有了细微区别的， 比如， 每个Pod的hostanme、名字等都是不同的， 携带了标红。 而statefulSet，区分这些实例的方式， 通过在Pod的名字里加上事先约定号的编号。
- 其次， **Kubernetes通过headless ， 为这些有编号的Pod， 在DNS服务器中生成带有同样标红的DNS记录**， 只有StatefulSet能够保证这些Pod的名字的编号不变， 那么Service里类似于 web-0.nginx.default.svc.cluster.local 这样的 DNS 记录也就不会变， 而这条记录解析出来的Pod的IP地址， 则会随着后端Pod的删除和再创建而自动更新， 这当然是Service机制本身的能力， 不需要StatefulSet操心
- 最后， **StatefulSet 还为每个Pod分配并创建一个同样编号的PVC**，， 这样 kubernetes 就可以通过Persistent Volume 机制为这个PVC绑定上毒药的PV， 从而保证了每个Pode都拥有一个独立的Volume

### 总结

StatefulSet 其实就是一种特殊的 Deployment，而其独特之处在于，它的每个 Pod 都被编号了。而且，这个编号会体现在 Pod 的名字和 hostname 等标识信息上，这不仅代表了 Pod 的创建顺序，也是 Pod 的重要网络标识（即：在整个集群里唯一的、可被的访问身份）。

有了这个编号后，StatefulSet 就使用 Kubernetes 里的两个标准功能：Headless Service 和 PV/PVC，实现了对 Pod 的拓扑状态和存储状态的维护。
