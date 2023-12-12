# Kubernetes Loki


# 日志聚合分析系统-Loki
loki 是Grafana的开源项目，是一个水平可扩展，高可用性，多租户的日志聚合系统，
为每个日志流编制一组标签，专门为 Prometheus 和 Kubernetes 用户做了相关优化

<!--more-->
## Loki的优势
* 不对日志进行全文索引，通过压缩非结构化日志和仅索引元数据
* 通过与Prometheus 相同的标签记录对日志进行索引和分组，
* 适合存储 Kubernetes Pod 日志，比如pod 标签之类的元数据会被自动删除和编入索引
* 受 Grafana 原生支持，与 Prometheus 配合更加方便

## 整体架构
{{< figure src="/images/loki01.png" title="loki01" >}}

![](media/17021090927978.jpg)

Loki 的组成部分
* Loki 为主服务器， 负责存储日志和处理查询
* Prometail 是代理， 负责手机日志并将其发送给Loki
* Grafana 用来UI展示。


Loki使用与Prometheus 相同的服务发现和标签重新标记库。编写了 Promtail。在 Kubernetes 中 Promtail 以 DaemonSet 方式运行在每个节点中，通过 Kubernetes API 得到日志的正确元数据，并将它们发送到Loki，如下图：

{{< figure src="/images/loki02.png" title="loki02" >}}

![](media/17021100059277.jpg)
Loki种主要组件有 DIstributor， Ingester和Querier

> 负责写入的组件有Distributor和Ingester两个
> 

{{< figure src="/images/loki03.png" title="loki03" >}}

![](media/17021104600500.jpg)

### Distributor
Promtaif 一旦将日志发给Loki， DIstributor就是第一个接收日志的组件。 由于 日志的写入量可能很大，所以不能再它传入时并行写入数据库， 要先进行批处理和压缩数据。
1. Distributor 接收到HTTP请求，用于存储流数据
2. 通过 hash 环对数据流进行hash
3. 通过 hash 算法计算出应该发送到哪个Ingester后， 发送数据流
4. Ingester新建Chunks或将数据追加到已有的Chunk上

{{< figure src="/images/loki04.png" title="loki04" >}}

![](media/17021116320056.jpg)

### Ingester

ingester 接收到日志开始构建 Chunk

{{< figure src="/images/loki05.png" title="loki05" >}}

![](media/17021116832070.jpg)
ingester 是一个有状态组件， 负责构建和刷新Chunk，当 Chunk达到一定数量或者时间后，刷新到存储中去， 每个流日志对应一个Ingester。index 和Chunk 各自使用单独的数据库， 因为他们存储额数据类型不同。

### 负责读的组件则是Querier：

{{< figure src="/images/loki06.png" title="loki06" >}}

![](media/17021120333900.jpg)

读取就 比较简单， 由Querier 负责 给定一个时间和标签选择器，也就是收到读请求：

1. Querier 收到HTTP读请求
2. Querier 将请求发送至Ingester读取还未写入Chunks的内存数据
3. 随后再去index+chunks中查找数据
4. Querier 遍历所有数据并进行去重处理，再返回最终结果


## 搭建使用

上边主要介绍的Loki的工作流程及组件，下面我们实际搭建操作下：

Loki项目地址：https://github.com/grafana/loki/

官网：https://grafana.com/oss/loki/

### 一、通过Helm部署：
```bash
## 添加chart
helm repo add loki https://grafana.github.io/loki/charts

## 更新chart
helm repo update 

## 将loki template下载到本地
helm fetch loki/loki-stack

## 解压并自定义修改参数
tar zxvf loki-stack-2.0.2.tgz
cd loki-stack/ $$ ls
charts  Chart.yaml  README.md  requirements.lock  requirements.yaml  templates  values.yaml

cd charts/ $$ ls
filebeat  fluent-bit  grafana  logstash  loki  prometheus  promtail
```

## 开始helm安装前需要注意几个点
1、可以修改values.yaml文件，指定是否开启Grafana、Prometheus等服务，默认不开启的：
```yaml
loki:
  enabled: true

promtail:
  enabled: true

fluent-bit:
  enabled: false

grafana:
  enabled: true
  sidecar:
    datasources:
      enabled: true
  image:
    tag: 6.7.0

prometheus:
  enabled: false
```

修改Grafana的values.yaml，使其Service暴露方式为NodePort(默认为ClusterIp)：
vim charts/grafana/values.yaml
```yaml
service:
  type: NodePort
  port: 80
  nodePort: 30002  # 端口范围：30000-32767
  targetPort: 3000
    # targetPort: 4181 To be used with a proxy extraContainer
  annotations: {}
  labels: {}
  portName: service
```

还有一处账号密码可以自定义修改下：
```yaml
# Administrator credentials when not using an existing secret (see below)
adminUser: admin
adminPassword: admin
```

## promtail服务在构建时会自动挂载

1. 宿主机docker 主目录下的 container目录，一般默认都为/var/lib/docker/containers
2. pod的日志目录， 一般默认在/var/log/pods

这就需要特别注意一下，如果是修改过docker默认的存储路径的，需要将mount的路径进行修改，promtail找不到对应的容器日志

具体docker 存储路径，可以使用docker info 命令查询
vim charts/promtail/values.yaml
```bash
# @default -- See `values.yaml`
defaultVolumes:
  - name: run
    hostPath:
      path: /run/promtail
  - name: containers
    hostPath:
      path: /var/lib/containers
  - name: pods
    hostPath:
      path: /var/log/pods

# -- Default volume mounts. Corresponds to `volumes`.
# @default -- See `values.yaml`
defaultVolumeMounts:
  - name: run
    mountPath: /run/promtail
  - name: containers
    mountPath: /var/lib/containers
    readOnly: true
  - name: pods
    mountPath: /var/log/pods
    readOnly: true
```

## 开始安装
```bash
volumes:
- name: docker
  hostPath:
    path: /data/lib/docker/containers  ## 我的是放在了data下
- name: pods
  hostPath:
    path: /var/log/pods

volumeMounts:
- name: docker
  mountPath: /data/lib/docker/containers  ## 挂载点也要进行修改
  readOnly: true
- name: pods
  mountPath: /var/log/pods
  readOnly: true
 

开始安装：

helm install -n loki --namespace loki -f values.yaml ../loki-stack
2020/11/11 17:18:54 Warning: Merging destination map for chart 'logstash'. The destination item 'filters' is a table and ignoring the source 'filters' as it has a non-table value of: <nil>
NAME:   loki
LAST DEPLOYED: Wed Nov 11 17:18:53 2020
NAMESPACE: loki
STATUS: DEPLOYED

RESOURCES:
==> v1/ClusterRole
NAME                       AGE
loki-promtail-clusterrole  1s
loki-grafana-clusterrole   1s

==> v1/ClusterRoleBinding
NAME                              AGE
loki-promtail-clusterrolebinding  1s
loki-grafana-clusterrolebinding   1s

==> v1/ConfigMap
NAME                  DATA  AGE
loki-grafana          1     1s
loki-grafana-test     1     1s
loki-loki-stack       1     1s
loki-loki-stack-test  1     1s
loki-promtail         1     1s

==> v1/DaemonSet
NAME           DESIRED  CURRENT  READY  UP-TO-DATE  AVAILABLE  NODE SELECTOR  AGE
loki-promtail  2        2        0      2           0          <none>         1s

==> v1/Deployment
NAME          READY  UP-TO-DATE  AVAILABLE  AGE
loki-grafana  0/1    1           0          1s

==> v1/Pod(related)
NAME                         READY  STATUS             RESTARTS  AGE
loki-0                       0/1    ContainerCreating  0         2s
loki-grafana-56bf5d8d-8zcgp  0/1    Init:0/1           0         2s
loki-promtail-6r24r          0/1    ContainerCreating  0         2s
loki-promtail-fvnfc          0/1    ContainerCreating  0         2s

==> v1/Role
NAME               AGE
loki-promtail      1s
loki-grafana-test  1s
loki               1s

==> v1/RoleBinding
NAME               AGE
loki-promtail      1s
loki-grafana-test  1s
loki               1s

==> v1/Secret
NAME          TYPE    DATA  AGE
loki          Opaque  1     1s
loki-grafana  Opaque  3     1s

==> v1/Service
NAME           TYPE       CLUSTER-IP      EXTERNAL-IP  PORT(S)       AGE
loki           ClusterIP  10.109.216.219  <none>       3100/TCP      1s
loki-grafana   NodePort   10.100.203.138  <none>       80:30002/TCP  1s
loki-headless  ClusterIP  None            <none>       3100/TCP      1s

==> v1/ServiceAccount
NAME               SECRETS  AGE
loki               1        1s
loki-grafana       1        1s
loki-grafana-test  1        1s
loki-promtail      1        1s

==> v1/StatefulSet
NAME  READY  AGE
loki  0/1    1s

==> v1beta1/PodSecurityPolicy
NAME               PRIV   CAPS      SELINUX           RUNASUSER  FSGROUP    SUPGROUP  READONLYROOTFS  VOLUMES
loki               false  RunAsAny  MustRunAsNonRoot  MustRunAs  MustRunAs  true      configMap,emptyDir,persistentVolumeClaim,secret,projected,downwardAPI
loki-grafana       false  RunAsAny  RunAsAny          RunAsAny   RunAsAny   false     configMap,emptyDir,projected,secret,downwardAPI,persistentVolumeClaim
loki-grafana-test  false  RunAsAny  RunAsAny          RunAsAny   RunAsAny   false     configMap,downwardAPI,emptyDir,projected,secret
loki-promtail      false  RunAsAny  RunAsAny          RunAsAny   RunAsAny   true      secret,configMap,hostPath,projected,downwardAPI,emptyDir

==> v1beta1/Role
NAME          AGE
loki-grafana  1s

==> v1beta1/RoleBinding
NAME          AGE
loki-grafana  1s
```

创建完成后，通过暴露的svc访问Grafana：
```bash
[root@Centos8 loki-stack]# kubectl get svc -n loki 
NAME            TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)        AGE
loki            ClusterIP   10.109.216.219   <none>        3100/TCP       113s
loki-grafana    NodePort    10.100.203.138   <none>        80:30002/TCP   113s
loki-headless   ClusterIP   None             <none>        3100/TCP       113s
```

## 开始使用Loki
通过服务器ip+30002 访问， 登陆成功， 
如果是安装Loki时采用的以上方法，开启了Grafana，那系统会自动配置好Data sources，应该不会有什么问题。

但是，如果是手动搭建的Grafana，需要手动添加Data Sources时，一定注意：

数据源名称中的Loki，L一定要是大写！！！

如果不是大写，会导致连接不到Loki源，一般回报错：Error connecting to datasource: Loki: Bad Gateway. 502

如果是Loki，L大写

{{< figure src="/images/loki07.png" title="loki07" >}}


![](media/17023498019869.jpg)

数据源添加完毕后，开始查看日志

点击Explore，可以看到选择labels的地方
{{< figure src="/images/loki08.png" title="loki08" >}}
![](media/17023499430391.jpg)

以下是labels的展现形式

{{< figure src="/images/loki09.png" title="loki09" >}}
![](media/17023499603207.jpg)

选择一个app:grafana的标签查看一下

{{< figure src="/images/loki10.png" title="loki10" >}}
![](media/17023499843419.jpg)

>  默认 Loki会将stdout（正常输出）类型和stderr（错误输出）类型全部展示出来

除了这种办法，还可以直接通过上边的搜索栏，进行自定义的筛选，具体的语法问题，可以再自行查询学习。

还可以查看 Prometheus 的 metrics 信息

{{< figure src="/images/loki11.png" title="loki11" >}}
![](media/17023505724162.jpg)

