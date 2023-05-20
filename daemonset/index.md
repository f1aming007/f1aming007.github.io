# Kubernets-DaemonSet


# DaemonSet

### 主要作用

在kubernetes集群里， 运行一个Daemon Pod，  所以， 这个Pod有如下三个特征

1. 这个Pod运载kubernetes集群里的每个节点（Node） 上
2. 每个节点上只有一个这样的Pod实例
3. 当有新的节点加入Kubernetes集群后， 该Pod 会自动的再新节点上被创建出来， 而当节点被删除后， 它上面的Pod也相应会被回收调。

### Daemon Pod 的例子

1. 各种网络插件的Agent组件， 都必须运行在每个节点上， 用来处理这个节点上的容器网络
2. 各种存储的插件的Agent组件， 也必须运行在每个节点上， 用来在这个节点上挂载远程存储目录，操作容器的Volume目录
3. 各种监控组件和日志组件， 也必须运行在每个节点上，复制这个节点上监控信息和日志搜索

# API

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluentd-elasticsearch
  namespace: kube-system
  labels:
    k8s-app: fluentd-logging
spec:
  selector:
    matchLabels:
      name: fluentd-elasticsearch
  template:
    metadata:
      labels:
        name: fluentd-elasticsearch
    spec:
      tolerations:
      - key: node-role.kubernetes.io/master
        effect: NoSchedule
      containers:
      - name: fluentd-elasticsearch
        image: k8s.gcr.io/fluentd-elasticsearch:1.20
        resources:
          limits:
            memory: 200Mi
          requests:
            cpu: 100m
            memory: 200Mi
        volumeMounts:
        - name: varlog
          mountPath: /var/log
        - name: varlibdockercontainers
          mountPath: /var/lib/docker/containers
          readOnly: true
      terminationGracePeriodSeconds: 30
      volumes:
      - name: varlog
        hostPath:
          path: /var/log
      - name: varlibdockercontainers
        hostPath:
          path: /var/lib/docker/containers
```

## Daemonset 如何确保每个Node上有且只有一个被管理的Pod？

Daemonset Controller ， 首先从Etcd 里获取所有的Node列表，然后遍历所有的Node， 这时， 它就可以很容器的去检查， 当前这个Node上是不是携带了name=fluentd-elasticsearch 标签的 Pod 在运行。

检查结果有三种情况

1. 没有每种Pod， 那么就意味着这个Node创建这样一个Pod
2. 有这种Pod， 但是数量大于1， 那就说明 多余的Pod从这个Node删除掉
3. 正好只有一个这种Pod， 说明这个节点是正常的

## nodeAffinity

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: with-node-affinity
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: metadata.name
            operator: In
            values:
            - node-geektime
```

### 含义

1. requiredDuringSchedulingIgnoredDuringExecution：这个nodeAffunuty 必须在每个调度的时候给予考虑， 同时 意味着可以设置在某个情况下不考虑这个nodeAffinity
2. 这个Pod, 将来只允许允许在“`metadata.name`是“node-geektime”的节点上

Daemonset Controller 会在创建Pod的时候， 自动在这个Pod的API对象里， 加上这样一个nodeAffinity定义

## tolerations

这个字段： 意味着这个Pod， 会“容忍”（Toleration）某些Node的污点（Taint）

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: with-toleration
spec:
  tolerations:
  - key: node.kubernetes.io/unschedulable
    operator: Exists
    effect: NoSchedule
```

### 含义

toleration： “容忍”所有被标记为unschedulable“污点”的Node， “容忍”的效果是允许调度

在 Kubernetes 项目中，当一个节点的网络插件尚未安装时，这个节点就会被自动加上名为`node.kubernetes.io/network-unavailable`的“污点”。

**而通过这样一个 Toleration，调度器在调度这个 Pod 的时候，就会忽略当前节点上的“污点”，从而成功地将网络插件的 Agent 组件调度到这台机器上启动起来。**
