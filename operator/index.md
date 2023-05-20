# Kubernets-Operator


# Operator 工作原理解读

# operator的工作原理和编写方法

### **第一步，将这个 Operator 的代码 Clone 到本地：**

```bash
git clone https://github.com/coreos/etcd-operator
```

## **第二步，将这个 Etcd Operator 部署在 Kubernetes 集群里**

```bash
example/rbac/create_role.sh
```

这个脚本为 Etcd Operator 创建 RBAC 规则， 因为 Etcd Operator 需要访问Kubernetes的APIServer来创建对象

1. 对Pod， service，PVC， Deployment， Secret等API对象， 有所有权限
2. 对CRD对象， 有所有权限
3. 对属于[etcd.database.coreos.com](http://etcd.database.coreos.com/) 这个 API Group 的 CR（Custom Resource）对象，有所有权限。

Etcd Operator 本身 是一个Deployment

```yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: etcd-operator
spec:
  replicas: 1
  template:
    metadata:
      labels:
        name: etcd-operator
    spec:
      containers:
      - name: etcd-operator
        image: quay.io/coreos/etcd-operator:v0.9.2
        command:
        - etcd-operator
        env:
        - name: MY_POD_NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
        - name: MY_POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
```

创建 Etcd Operator

```yaml
kubectl create -f example/deployment.yaml
```

pod进入Running 状态， 有一个CRD被自动创建出来

```yaml
$ kubectl get pods
NAME                              READY     STATUS      RESTARTS   AGE
etcd-operator-649dbdb5cb-bzfzp    1/1       Running     0          20s
 
$ kubectl get crd
NAME                                    CREATED AT
etcdclusters.etcd.database.coreos.com   2018-09-18T11:42:55Z
```

通过 kubectl describe 命令看到它的细节，如下所示：

```yaml
$ kubectl describe crd  etcdclusters.etcd.database.coreos.com
...
Group:   etcd.database.coreos.com
  Names:
    Kind:       EtcdCluster
    List Kind:  EtcdClusterList
    Plural:     etcdclusters
    Short Names:
      etcd
    Singular:  etcdcluster
  Scope:       Namespaced
  Version:     v1beta2
  
...
```

## 编写EtcdClutser的Yaml，

```yaml
$ kubectl apply -f example/example-etcd-cluster.yaml
```

这个 example-etcd-cluster.yaml 文件里描述的，是一个 3 个节点的 Etcd 集群

```yaml
kubectl get pods
NAME                            READY     STATUS    RESTARTS   AGE
example-etcd-cluster-dp8nqtjznc   1/1       Running     0          1m
example-etcd-cluster-mbzlg6sd56   1/1       Running     0          2m
example-etcd-cluster-v6v6s6stxd   1/1       Running     0          2m
```

```yaml
apiVersion: "etcd.database.coreos.com/v1beta2"
kind: "EtcdCluster"
metadata:
  name: "example-etcd-cluster"
spec:
  size: 3
  version: "3.2.13"
```

## Operator 的工作原理：

实际上利用了Kubernetes的自定义API资源（CRD），来描述我们想要不熟的“有状态应用”，然后再自定义控制器里， 根据自定义API对象的变化， 来完成具体的部署和运维工作

tcd Operator 在业务逻辑的实现方式上

![](media/16845535981861.jpg)


**第一个工作只在该CLuster对象第一次被创建的时候会执行， 这个工作， 就是我们前面提到Bootstrap，即：创建一个单节点的种子集群。**

**Bootstrap，即：创建一个单节点的种子集群。**
