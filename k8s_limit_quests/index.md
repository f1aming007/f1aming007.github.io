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

## CPU的特殊性

## 内存的特殊性

## Namespace ResourceQuta

## Namespace LimitRange

