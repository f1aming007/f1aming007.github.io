# Kubernetes_核心组件工作原理


# 图解 Kubernetes 核心组件工作原理

## 问题1: Master节点和Worker节点如何通信？

首先， 当Master节点启动时， 会运行一个 Kube-apiserver进程，它提供了集群管理的API接口，是集群中各个功能模块之间进行数据交互和通信的中心枢纽， 同时也提供了一个完善的机器安全机制。

在Node 节点上， 利用Kubernetes中的kubelet组件，每个Node节点上都会运行一个kubelet进程，负责向Master 汇报本节点的运行状态， 如 Node节点注册、终止、定期健康报告等等，并接收来自Master 的命令并创建相应的Pod。

在`Kubernetes中， Pod是最基本的运行单元`，它与Docker容器略有不同， 因为pod中可能包好一个或者多个容器， 这些容器内部共享网络资源，即可以通过localhost 相互访问

关于如何在Pod中实现网络共享，每个pod启动，内部都会启动一个pause容器，它使用默认的网络模式，其他容器的网络设置完成网络共享问题

{{< figure src="/images/k8s01.png" title="k8s01" >}}
![](media/17048741105220.jpg)


## 问题2: Master 如何将Pod调度到指定的Node上？

这项工作由 Kube-scheduler完成，整个调度过程通过执行一系列复杂的算法，最终为每个pod计算出一个最优的目标Node， 这是由Kube-scheduler进程自动完成的

`最常见的是循环调度（RR）`。当然也有可能需要将Pod调度到指定的Node上，我们可以通过将节点的标签（Label）与Pod的节点选择器进行匹配来达到指定的效果

{{< figure src="/images/k8s02.png" title="k8s02" >}}

![](media/17048754948493.jpg)

## 问题3: 各个节点和Pod的信息统一在哪里维护，谁来维护？

从上面的Pod调度来看， 我们必须有一个存储中心来存储每个节点的每个Pod的资源使用情况、监控状态和基本情况， 这样Pod调度才能正常进行

`在Kubernetes中，etcd组件被用作高可用和一致的存储库` 该组件可以内置在Kubernets中， 也可以外部构建提供Kubernetes使用。

`集群上的所有配置信息都存储在etcd中`， 考虑到各个组件的相对应的独立性和整体的可维护性，这些存储的数据的增删改查都是统一由 Kube-apiserver来调用的，并且apiserver还提供了REST支持， 不仅为各个内部组件提供服务单也想集群外部的用户公开服务

外部用户可以通过REST接口或kubectl命令行工具管理集群，该工具内部与apiserver通信

{{< figure src="/images/k8s03.png" title="k8s03" >}}

![](media/17048776415289.jpg)

## 问题4: 外部用户如何访问集群中的运行的pod？

`在分布式集群中， 服务往往由多个应用提供，以分担访问压力，而这些应用可能分布在多个节点上， 这就设计到跨主机通信`

这里Kubernetes引入了Service的概念，将多个相同的Pod包装成一个完整的服务，对外提供服务。至于获取这些相同的Pod，每个Pod在启动时都会设置labels为attribute。

在服务中， 我们传递选择器Selectort， 选择与整体服务具有相同Name标签属性的Pod， 将服务信息通过Apiserver存储到etcd中， 由Service Controller完成，同时在每个节点上启动一个kube-proxy进程， 负责从服务地址到Pod地址的代理和负载均衡

{{< figure src="/images/k8s04.png" title="k8s04" >}}

![](media/17049408905683.jpg)

## 问题5: Pod 如何动态扩容和伸缩？
在Kubernetes 中，使用Replication Controller进行管理，为每个Pod设置一个预期的副本数，当实际副本数量不符合预期时，动态调整数量以达到预期值，所需值可以由我们手动更新，也可以由自动缩放代理更新

{{< figure src="/images/k8s05.png" title="k8s05" >}}

![](media/17049425021996.jpg)



## 问题6:各个组件如何协同工作？
Kube-controller-manager进程的作用。我们知道ectd是作为集群数据的存储中心，而apiserver是用来管理数据中心，充当其他进程与数据中心通信的桥梁。

Service Controller 和 Replication Controller由Kube-controller-manager管理。`作为一个守护进程，每个Controller都是一个控制回路， 通过apiserver监控集群的共享状态，并尝试改变 将实际状态中不合符预期的变化`
关于 Controller，管理器还包含 Node Controller、 ResourceQuota Controller、Namespace Controller 等

{{< figure src="/images/k8s06.png" title="k8s06" >}}

![](media/17049428233678.jpg)

