# K8s_limit_quests

![limit](https://sysdig.com/wp-content/uploads/Kubernetes-Limits-and-Request-01.png)
# Kubernetes 的 Limits 和 Requests
在k8s适应容器时， 要知道所涉及的资源时什么以及如何需要他们，有些进程比其他进程需要更多的CPU和内存
知道这些之后， 才能正确的配置我们的容器和pod
* Kubernetes 的Limits和Requests介绍
* 实践案例
* Kubernetes Requests
* Kubernetes Limits
* CPU的特殊性
* 内存的特殊性
* Namespace ResourceQuta
* Namespace LimitRange
* 总结

<!--more-->


## Kubernetes 的Limits和Requests介绍
在适应Kubernetes 时，Limits和Requests 时重要的配置，主要包含了CPU和内存的配置

kubernetes将Limits定义为一个容器使用的最大资源量，意味着容器的消耗量永远不能超过显示的内存或CPU量。

Requests时指容器保留的最小保证量
![Requests](https://sysdig.com/wp-content/uploads/Kubernetes-Limits-and-Request-04-1-1170x585.png)

## 实践案例
deployment - demo
```yaml
kind: Deployment
apiVersion: extensions/v1beta1
…
template:
  spec:
    containers:
      - name: redis
        image: redis:5.0.3-alpine
        resources:
          limits:
            memory: 600Mi
            cpu: 1
          requests:
            memory: 300Mi
            cpu: 500m
      - name: busybox
        image: busybox:1.28
        resources:
          limits:
            memory: 200Mi
            cpu: 300m
          requests:
            memory: 100Mi
            cpu: 100m
```
将deployment 部署到4C16 G配置的节点上， 可以得到信息
![](https://sysdig.com/wp-content/uploads/Kubernetes-Limits-and-Request-05.png)

1. Pod的有效请求时400MiB的内存和600 millicores的CPU， 需要一个有足够资源的可分配空间的节点来安排pod
2. Redis容器的CPU份额将是512， 而busybox容器时102，Kubernetes总是为每个核心分配的1024个份额，因此redis：1024 * 0.5 cores ≅ 512和busybox：1024 * 0.1核 ≅ 102
3. 如果Redis容器试图分配超过600MB的RAM，它将被OOM杀死，很可能是pod失败
4. 如果Redis试图在每100ms内使用超过100ms的CPU， （因为有4个CPU， 可用时间为每个100ms 400ms），它将遭受CPU节流，导致性能下降
5. 如果Busybox容器试图分配超过200MB的RAM，它将被OOM杀死，导致一个失败的Pod
6. 如果Busybox试图每100ms使用超过30ms的CPU，它将遭受CPU节流，导致性能下降。

## Kubernetes Requests
Kubernetes 将请求定义为容器使用的资源最低保证量

基本上，他将设定容器所需消耗的资源的最小数量

当一个pod被调度时，kube-scheduler将检查kebernetes请求，以便将其分配给一个特定的节点，该节点至少可以满足pod中所有容器的这个数量，如果请求的数量高于可用的资源，pod将不会被安排，并保持在Pending状态

在这个例子中，在容器定义中，我们设置了一个请求，要求100m核心的CPU和4Mi的内存

```yaml
resources:
   requests:
        cpu: 0.1
        memory: 4Mi
```
Requests通常被使用在一下场景：
* 当把Pod分配给一个节点时， 所以Pod中的容器的指定请求被满足
* 在运行时， 指定的请求量被保证在该Pod 中的容器的最小值。
![](https://sysdig.com/wp-content/uploads/image4.png)

## Kubernetes Limits
Kubernetes 将Limits定义为一个容器使用的最大资源量
这意味着容器的消耗量永远不能超过指定的内存量或CPU量
```yaml
resources:
      limits:
        cpu: 0.5
        memory: 100Mi
```
Limits通常用于一下场景：
* 当把Pod分配给一个节点时，如果没有设置请求，默认情况下，Kubernetes将分配请求=限制
* 在运行时， Kubernetes 将检查Pod中的容器所消耗的资源量是否过于限制所显示的数量

![](https://mmbiz.qpic.cn/mmbiz_jpg/iaqrvickekLVQtpcicU66M5NrHgs8ib7NDHIibIs6HKDOWLmV6oO5L44y267jWUB5oDEEmXqolOGSoic8q4HD6wpshvQ/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)


## CPU的特殊性
CPU是一种可压缩的资源，这意味他可以被拉伸，一满足所有的需求，如果进程要求的太多的CPU，其中一些将被节制

CPU代表计算处理时间，以核为单位
* 你可以用毫微米（m）来表示比一个核心更小的数量（例如，500米是半个核心）
* 最小的数量时1m
* 一个节点可能有一个以上的核心可用， 所以请求CPU>1时kennel的

![](https://mmbiz.qpic.cn/mmbiz_jpg/iaqrvickekLVQtpcicU66M5NrHgs8ib7NDHICGD30ZXm7Ty8T0h3H2U9vTg3MHJicYm2avEcqQC0uvTDmWWdWIN2bCw/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)


## 内存的特殊性
内存时一种不可压缩的资源，意味着它不能像CPU那样被拉伸，如果一个进程没有得到足够的内存来工作， 这个进程就会被杀死

在Kubernetes中，内存的单位是字节。
* 你可以用，E，P，T，G，M，k来代表Exabyte，Petabyte，Terabyte，Gigabyte，Megabyte和kilobyte，尽管只有最后四个是常用的。(例如，500M, 4G)
* 不要用小写的m表示内存（这代表Millibytes，低得离谱）
* 你可以用Mi来定义Mebibytes，其余的也可以用Ei、Pi、Ti来定义（例如，500Mi）

![](https://mmbiz.qpic.cn/mmbiz_jpg/iaqrvickekLVQtpcicU66M5NrHgs8ib7NDHI3icibXBHv8yNorck4gbm1kh4KYA5UvLic0Y4zbeMrwWhJFbDSxHj6Zv7A/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

## 最佳实践
在Kubernetes中， 你应该很少使用限制来控制你的资源的使用，这是因为如果你想避免使用饥饿（确保每个重要的进程都能得到它的份额）， 应该首先使用请求

通过设置限制，你只是防止进程在特殊情况下检索额外的资源，在内存方面造成OOM杀戮，在CPU方面造成Throttling（进程将需要等待CPU可以再次使用）

如果一个Pod的所有请求中设置一个等于限制的请求值，该Pod将获得保证的服务质量

还需要注意的是， 资源使用量高于请求的Pod更有可能被驱逐，所以设置非常低的请求会造成弊大于利。

## Namespace ResourceQuta
由于命名空间的存在，我们可以将Kubernetes资源隔离到不同的组，也称为租户

通过ResourceQuota， 可以为整个命令空间设置一个内存或CPU限制，确保其中的实体不能消耗超过这个数量
```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: mem-cpu-demo
spec:
  hard:
    requests.cpu: 2
    requests.memory: 1Gi
    limits.cpu: 3
    limits.memory: 2Gi
```

* requests.cpu：这个命名空间中所有请求的最大CPU数量。
* requests.memory：这个命名空间中所有请求的最大内存量。
* limits.cpu：这个命名空间中所有限制的最大CPU数量。
* limits.memory：这个命名空间中所有限制的总和的最大内存量。

## Namespace LimitRange
如果我们想限制一个命名空间可分配的资源总量，ResourceQuotas很有用。但如果我们想给里面的元素提供默认值，会发生什么？

LimitRanges是一种Kubernetes策略，它限制了命名空间中每个实体的资源设置。
```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: cpu-resource-constraint
spec:
  limits:
  - default:
      cpu: 500m
    defaultRequest:
      cpu: 500m
    min:
      cpu: 100m
    max:
      cpu: "1"
    type: Container
```
* default。如果没有指定，创建的容器将有这个值。
* min: 创建的容器不能有比这更小的限制或请求。
* max: 创建的容器不能有大于此值的限制或请求。

以后， 如果创建一个没有设置请求或限制的新的Pod， LimitRange会自动为其所有的容器设置这些值。
```yaml
 Limits:
      cpu:  500m
    Requests:
      cpu:  100m
```

## 总结
kubernetes集群选择最佳的限制是关键，以便获得最佳的能源消耗和成本

为我们的Pod分配过多的资源可能会导致成本激增。

规模过小或专用于极少的CPU或内存将导致应用程序不能正常运行，甚至Pod被驱逐。

## 参考文档
【1】https://sysdig.com/blog/kubernetes-pod-pending-problems/
【2】https://sysdig.com/blog/troubleshoot-kubernetes-oom/
【3】https://sysdig.com/blog/kubernetes-pod-evicted/
原文：https://sysdig.com/blog/kubernetes-limits-requests/
