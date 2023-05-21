# Kubernets-Sc&Pv&Pvc


{{< figure src="/images/featured-image.jpg" title="Lighthouse (figure)" >}}
# **PV、PVC、StorageClass**

Kubernetes 处理容器持久化存储的核心原理


# PV： 持久化存储数据卷

pv 一般有运维人员事先创建之后使用， 定义一个NFS类型的PV

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: nfs
spec:
  storageClassName: manual
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteMany
  nfs:
    server: 10.244.1.4
    path: "/"
```

# PVC： pod所希望使用的持久化存储的属性

一般有开发人员创建， Volume存储的大小，可读写的权限

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: nfs
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: manual
  resources:
    requests:
      storage: 1Gi
```

## PVC与PV绑定使用的条件

1. **PV和PVC的spec字段，PV的存储大小， 就必须满足PVC的要求**
2. **PV 和 PVC 的 storageClassName 字段必须一样**

YAML 文件里声明使用这个 PVC 了

```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    role: web-frontend
spec:
  containers:
  - name: web
    image: nginx
    ports:
      - name: web
        containerPort: 80
    volumeMounts:
        - name: nfs
          mountPath: "/usr/share/nginx/html"
  volumes:
  - name: nfs
    persistentVolumeClaim:
      claimName: nfs
```

## PersistentVolumeController

专门处理持久化存储的控制器， 会不断地查看当前每一个 PVC，是不是已经处于 Bound（已绑定）状态。如果不是，那它就会遍历所有的、可用的 PV，并尝试将其与这个“单身”的 PVC 进行绑定

**所谓容器的 Volume，其实就是将一个宿主机上的目录，跟一个容器里的目录绑定挂载在了一起**

**而所谓的“持久化 Volume”，指的就是这个宿主机上的目录，具备“持久性”**

**这个准备“持久化”宿主机目录的过程，我们可以形象地称为“两阶段处理**

一个Pod调度到一个节点上之后， kubelet就要负责这个Pod创建的它的Volume目录，默认这个情况下，kubelet 为 Volume 创建的目录是如下所示的一个宿主机上的路径：

```yaml
/var/lib/kubelet/pods/<Pod 的 ID>/volumes/kubernetes.io~<Volume 类型 >/<Volume 名字 >
```

1. 这一步**为虚拟机挂载远程磁盘的操作，对应的正是“两阶段处理”的第一阶段。在 Kubernetes 中，我们把这个阶段称为 Attach**
2. **将磁盘设备格式化并挂载到 Volume 宿主机目录的操作，对应的正是“两阶段处理”的第二个阶段，我们一般称为：Mount。**

在这一步，kubelet 需要作为 client，将远端 NFS 服务器的目录（比如：“/”目录），挂载到 Volume 的宿主机目录上，即相当于执行如下所示的命令：

```yaml
$ mount -t nfs <NFS 服务器地址 >:/ /var/lib/kubelet/pods/<Pod 的 ID>/volumes/kubernetes.io~<Volume 类型 >/<Volume 名字 >
```

kubelet 只要把这个 Volume 目录通过 CRI 里的 Mounts 参数，传递给 Docker，然后就可以为 Pod 里的容器挂载这个“持久化”的 Volume 了。其实，这一步相当于执行了如下所示的命令

```yaml
$ docker run -v /var/lib/kubelet/pods/<Pod 的 ID>/volumes/kubernetes.io~<Volume 类型 >/<Volume 名字 >:/< 容器内的目标目录 > 我的镜像 ...
```

**PV 的“两阶段处理”流程，是靠独立于 kubelet 主控制循环（Kubelet Sync Loop）之外的两个控制循环来实现的**

# StorageClass

k8s提供了一套自动创建PV的机制， Dynamic Provisioning

**storageClass对象的作用， 其实就是创建PV的模板**

主要定义两部分

1. PV的属性， （存储类型， Volume的大小）
2. 创建这种PV需要用到的存储插件， （nfs、Ceph）

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: block-service
provisioner: kubernetes.io/gce-pd
parameters:
  type: pd-ssd
```

```yaml
apiVersion: ceph.rook.io/v1beta1
kind: Pool
metadata:
  name: replicapool
  namespace: rook-ceph
spec:
  replicated:
    size: 3
---
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: block-service
provisioner: ceph.rook.io/block
parameters:
  pool: replicapool
  #The value of "clusterNamespace" MUST be the same as the one in which your rook cluster exist
  clusterNamespace: rook-ceph
```

{{< figure src="/images/16845536761329.jpg" title="Lighthouse (figure)" >}}

![](media/16845536761329.jpg)


- PVC描述的， 是Pod想要使用的持久化存储的属性（存储的大小、读写权限）
- PV的描述， 一个具体的Volume的属性， （Volume的类型， 挂载目录。 远程存储服务地址）
- StorageClass 的作用， 充当PV的模板， 只有属于一个StorageClass的PV和PVC才可以绑定子啊一起
