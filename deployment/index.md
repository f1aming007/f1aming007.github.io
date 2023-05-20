# Kubernets-Deployment

# Deployment

### Deployment

- **Pod 的 “水平扩展、收缩”**

### ReplicaSet

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: nginx-set
  labels:
    app: nginx
spec:
  replicas: 3
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
        image: nginx:1.7.9
```

**一个ReplicaSet对象， 其实就是由副本数目的定义和一个Pod模板组合，** 

**更重要的是 Deployment控制器实际操纵的， 正是ReplicasSet对象， 而不是Pod对象**

### Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 3
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
        image: nginx:1.7.9
        ports:
        - containerPort: 80
```

### Deployment 、 ReplicaSet、 Pod的关系

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/578b220b-19f1-4839-af2b-9d3ec214e343/Untitled.png)

### 水平扩展、收缩的操作

```yaml
$ kubectl scale deployment nginx-deployment --replicas=4
deployment.apps/nginx-deployment scaled
```

### 滚动更新

```yaml
$ kubectl create -f nginx-deployment.yaml --record
```

-record 参数： 记录每次操作的执行命令， 方便后面查看

检查一下 nginx-deployment 创建后的状态信息

```yaml
$ kubectl get deployments
NAME               DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
nginx-deployment   3         0         0            0           1s
```

状态含义

1. DESIERD：用户期望的Pod 副本个数
2. CURRENT：当前处于Running状态的Pod的个数
3. UP-TO-DATE： 当前处于最新版的Pod的个数，  （Pod的Spec部分与Deployment的Pod模板的定义的一致）
4. AVALABLE： 当前已经可用的Pod的个数， 既是 Running 状态，又是最新版本，并且已经处于 Ready（健康检查正确）状态的 Pod 的个数。

查看Deployment对象的状态变化kubectl rollout status

```yaml
$ kubectl rollout status deployment/nginx-deployment
Waiting for rollout to finish: 2 out of 3 new replicas have been updated...
deployment.apps/nginx-deployment successfully rolled out
```

### Deployment对应进行版本控制的具体原理

这个镜像名字修改成为了一个错误的名字，比如：nginx:1.91。这样，这个 Deployment 就会出现一个升级失败的版本。

```yaml
$ kubectl set image deployment/nginx-deployment nginx=nginx:1.91
deployment.extensions/nginx-deployment image updated
```

由于这个 nginx:1.91 镜像在 Docker Hub 中并不存在，所以这个 Deployment 的“滚动更新”被触发后，会立刻报错并停止。

这时，我们来检查一下 ReplicaSet 的状态，如下所示：

```yaml
$ kubectl get rs
NAME                          DESIRED   CURRENT   READY   AGE
nginx-deployment-1764197365   2         2         2       24s
nginx-deployment-3167673210   0         0         0       35s
nginx-deployment-2156724341   2         2         0       7s
```

回滚到一起的旧版本

```yaml
$ kubectl rollout undo deployment/nginx-deployment
deployment.extensions/nginx-deployment
```

**要使用 kubectl rollout history 命令，查看每次 Deployment 变更对应的版本**

```bash
$ kubectl rollout history deployment/nginx-deployment
deployments "nginx-deployment"
REVISION    CHANGE-CAUSE
1           kubectl create -f nginx-deployment.yaml --record
2           kubectl edit deployment/nginx-deployment
3           kubectl set image deployment/nginx-deployment nginx=nginx:1.91
```

查看每个版本对应的Deployment的API对象的细节

```bash
$ kubectl rollout history deployment/nginx-deployment --revision=2
```

可以在kubectl roolout undo 命令加上回滚的版本号， 指定版本回滚

```bash
$ kubectl rollout undo deployment/nginx-deployment --to-revision=2
deployment.extensions/nginx-deployment
```

对Deployment的多次更新操作， 最后只生成一个ReplicaSet，

```bash
$ kubectl rollout pause deployment/nginx-deployment
deployment.extensions/nginx-deployment paused
```

是Deployment进入一个暂定状态，  可以随意使用 kubectl edit 或者 kubectl set image 指令，修改这个 Deployment 的内容了。

操作完成之后将Deployment 恢复回来

```bash
$ kubectl rollout resume deploy/nginx-deployment
deployment.extensions/nginx-deployment resumed
```

### 如何控制“历史的” ReplicaSet数量

Deployment 对象有一个字段，叫作 spec.revisionHistoryLimit，就是 Kubernetes 为 Deployment 保留的“历史版本”个数。所以，如果把它设置为 0，你就再也不能做回滚操作了

