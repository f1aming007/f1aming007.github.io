# Kubernets-声明式API


# 声明式API与Kubernetes编码范式

kubectl create 再replace的操作， 称为命令式配置文件操作

# 声明式API

[https://hugbz2.51cg3.co/archives/25451](https://hugbz2.51cg3.co/archives/25451/)

[/](https://hugbz2.51cg3.co/archives/25451/)[https://hugbz2.51cg3.co/archives/4081/](https://hugbz2.51cg3.co/archives/4081/)

kuberctl apply 命令就是 声明式API

### apply 与 replace命令的本质区别

- replace： 是使用新的YAML文件中的API对象，替换原来的API对象
- apply： 执行了一个对原有的API对象的PATCH操作

对于kube-apiserver在响应命令式请求的时候

- replace： 只能处理一个写请求， 否则会有产生冲突的可能
- apply：    **一次能处理多个写操作， 并且具备Merge能力**

# Istio

![](media/16845537847763.jpg)


Istio最根本的组件， 是运行在每一个应用Pod里的Envoy容器

Istio项目， 代理服务以sidecar容器的方式， 运行在每一个被治理的应用Pod中，

Envoy容器就能够通过配置Pod里的iptables规则， 把整个Pod的进出流量接管下来

Istio 的控制层里的Pilot组件，就能够调用每个Envoy容器的API， 对这个Envoy代理进行配置， 从而实现微服务治理。

### Istio项目使用的， 是kubernetes中一个非常重要的功能Dynamic Adminssion Control

Kubernetes 项目为我们额外提供了一种“热插拔”式的 Admission 机制，它就是 Dynamic Admission Control，也叫作：Initializer。

现在，我给你举个例子。比如，我有如下所示的一个应用 Pod：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
  labels:
    app: myapp
spec:
  containers:
  - name: myapp-container
    image: busybox
    command: ['sh', '-c', 'echo Hello Kubernetes! && sleep 3600']
```

Istio项目要做的， 就是在这个Pod Yaml被提交给kubernetes之后， 它对应的API对象字段加上Envoy容器的配置， 

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
  labels:
    app: myapp
spec:
  containers:
  - name: myapp-container
    image: busybox
    command: ['sh', '-c', 'echo Hello Kubernetes! && sleep 3600']
  - name: envoy
    image: lyft/envoy:845747b88f102c0fd262ab234308e9e22f693a1
    command: ["/usr/local/bin/envoy"]
    ...
```

Istio 要做的，就是编写一个用来为 Pod“自动注入”Envoy 容器的 Initializer。

首先， Istio会将这个Envoy容器本身的定义， 以ConfigMap方式保存在Kubernetes当前

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: envoy-initializer
data:
  config: |
    containers:
      - name: envoy
        image: lyft/envoy:845747db88f102c0fd262ab234308e9e22f693a1
        command: ["/usr/local/bin/envoy"]
        args:
          - "--concurrency 4"
          - "--config-path /etc/envoy/envoy.json"
          - "--mode serve"
        ports:
          - containerPort: 80
            protocol: TCP
        resources:
          limits:
            cpu: "1000m"
            memory: "512Mi"
          requests:
            cpu: "100m"
            memory: "64Mi"
        volumeMounts:
          - name: envoy-conf
            mountPath: /etc/envoy
    volumes:
      - name: envoy-conf
        configMap:
          name: envoy
```

Initializer 更新用户的P大对象的时候， 必须使用PATCH API 来完成， 而这种PATCH API 正式声明式API的主要能力

Istio 将一个编写好的Initializer， 作为一个Pod部署在kubernetes中

```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    app: envoy-initializer
  name: envoy-initializer
spec:
  containers:
    - name: envoy-initializer
      image: envoy-initializer:0.0.1
      imagePullPolicy: Always
```

不断获取“实际状态”， 然后“期望状态”做对比

initializer的控制器，  

- 不断获取的“实际状态”用户新创建的Pod
- “期望状态” 就是破的被添加的Envoy容器的定义

**有了这个 TwoWayMergePatch 之后，Initializer 的代码就可以使用这个 patch 的数据，调用 Kubernetes 的 Client，发起一个 PATCH 请求**

Istio项目的核心， 就是由无数个运行在应用Pod中的Envoy容器组成的服务代理网格。

## kubernetes “声明式API”的独到之处

---

- 首先， 所谓“声明式”， 我们定义个定义好的API对象来“声明”， 所期望的状态是什么样子的
- 其次， “声明式API”允许有多个API写端， 以PATCH的方式对API对象进行修改，而无需关心本地原始的YAML文件的内容
- 最后， Kubernetes项目才可以基于对API对象的增、删、该、查， 在完全无需外界干扰的情况下， 完成对“实际状态”和“期望状态”的协调过程

## Kubernetes 编程范式

> **如何使用控制器模式，同Kubernetes里API对象“增、删、改、查”进行协作，完成用户业务逻辑的编写过程**
> 

## 为 Network 这个自定义 API 对象编写一个自定义控制器（Custom Controller）

你好， 监控看到15、16、17这三台机器资源使用率和负载都太不高。 

目前现在我们这边资源需求较多，计划3月8号（下周三）回收这3台GPU机器。请知悉。

## 自定义控制器的原理

![](media/16845538063447.jpg)


**控制器的工作流程**

1. 从kubernetes的APISerer里获取它所关心的对象， 就是自定义的Network
2. informer与API 对象时——对应的， 所以我传递给自定义控制器的， 是Network对象informer
3. Network Informer跟APIServer建立连接， 是informer所使用的Reflector包
4. Reflector 使用的ListAndWatch的方法，来“获取”并“监听”这些Network对象实例的变化
5. 在ListAndWatch机制下， 一旦APIServer端有新的Network实例被创建、删除或者更新Reflector都会收到“事件通知”， 会放进一个Delta FIFO Queue（增量先进先出队列）中
6. informe会不断从这个Delta FIFO Queue 读取增量，  每拿到一个增量， informer 判断这个增量的事件类型。 然后创建或者更新本地对象的缓存。（这个缓存在kubernetes里叫Store）

**informer职责**

1. 同步本地缓存的工作
2. 根据这些事件的类型，触发事先注册号的**ResourceEventHandler**

informer 是一个钓友本地缓存和索引机制的 可以注册的EventHandler的client
