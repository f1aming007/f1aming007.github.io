# K8s GVK_GVR

# K8s-GVK&FVR
![Minion](https://images.unsplash.com/photo-1667372459510-55b5e2087cd0?ixlib=rb-4.0.3&ixid=M3wxMjA3fDB8MHxwaG90by1wYWdlfHx8fGVufDB8fHx8fA%3D%3D&auto=format&fit=crop&w=1932&q=80)
## GVK & GVR 介绍
* GVK —— group 、version、 kind
* GVR —— group 、version 、resource

<!--more-->
### kind 和 resource 概念

* 在编码过程中， 资源数据存储都是**结构体存储**
    * 多版本version存储在（alpha1，beta1，v1等），不同版本中存储结构体的存在着差异
    * 用 Kind 名（如 Deployment），并不能准确获取到其使用哪个版本结构体
    * 采用 GVK 获取到一个具体的 存储结构体，也就是 GVK 的三个信息（group/verion/kind) 确定一个 Go type（结构体）
    * Scheme 存储了 GVK 和 Go type 的映射关系
* 创建资源， 编写yaml，提交请求
    * 编写 yaml 过程中，我们会写 apiversion 和 kind，其实就是 GVK
    * 与 apiserver 通信是 http 形式，就是将请求发送到某一 http path
    * http path 其实就是 GVR
        * /apis/batch/v1/namespaces/default/job 这个就是表示 default 命名空间的 job 资源 

## 相同名称 Kind 可以存在不同组
相同名称的kind不仅可以存在不同的Version， 也可以存在不同的group
* Ingress, NetworkPolicy同时在这个两个API Group： extensions、 networking.k8s.io
* Deployment, DaemonSet, ReplicaSet同时在这些API Group中：extensions、 apps
* Event同时在这些API Group中： core group and events.k8s.io

## API-group 资源分组
* 各组可以单独打开或者关闭
* 各组可以有自己独立的版本， 不影响其他组的情况下可以单独衍化
* 同一个资源可以同时存在于多个不同组中，这样就可以同时支持某个特定资源稳定版本与实验版本

> kubernetes API 
> 查看当前 kubernetes 集群支持的 API 版本
```bash
$ kubectl api-versions
apiextensions.k8s.io/v1beta1
apiregistration.k8s.io/v1beta1
apps/v1beta1
apps/v1beta2
authentication.k8s.io/v1
authentication.k8s.io/v1beta1
authorization.k8s.io/v1
authorization.k8s.io/v1beta1
autoscaling/v1
autoscaling/v2beta1
batch/v1
batch/v1beta1
certificates.k8s.io/v1beta1
custom-metrics.metrics.k8s.io/v1alpha1
extensions/v1beta1
monitoring.coreos.com/v1
networking.k8s.io/v1
policy/v1beta1
rbac.authorization.k8s.io/v1
rbac.authorization.k8s.io/v1beta1
storage.k8s.io/v1
storage.k8s.io/v1beta1
v1
```

* 对于同一个资源对象的不同版本，API-Server 负责不同版本之间的无损切换，这点对于客户端来说是完全透明的（无感知）。
* 事实上，不同版本的同类型的资源在持久化层的数据可能是相同的。
    * 例如，对于同一种资源类型支持 v1 和 v1beta1 两个 API 版本，以 v1beta1 版本创建该资源的对象，后续可以以v1 或者 v1beta1 来更新或者删除该资源对象
    
## GVK 与 GVR 映射
* **kind** 是API <u>顶级</u> 资源对象类型，每个资源对象都需要kind来区分自身代表的类型
    * kind 字段 该资源对象的类型
        * 单个的资源对象类型（pod）
        * 资源对象列表类型， （podlist 或者 nodelist）
        * 特殊类型以及非吃就好类型 （很多这种类型的资源是 subresource， 例如用于绑定资源的 /binding、更新资源状态的 /status 以及读写资源实例数量的 /scale）

* **Resource**  通过http协议以JSON格式发送或者读取的资源展现形式
    * 单个资源 （.../namespaces/default）
    * 列表资源（.../jobs）

GVR 常用于组合成 RESTful API 请求路径
```bash
GET /apis/apps/v1/namespaces/{namespace}/deployments/{name}
```

## API 存储
*  API-Server 是无状态的，它需要与分布式存储系统 etcd 交互来实现资源对象的持久化操作
*  Kubernetes 资源对象是以JSON或  Protocol Buffers 格式存储在 etcd 中
    *  以通过配置 kube-apiserver 的启动参数 --storage-media-type 来决定想要序列化数据存入 etcd 的格式

* 创建一个 pod，然后使用 etcdctl 工具来查看存储在 etcd 中数据
```bash
$ cat << EOF | kubectl create -f -
apiVersion: v1
kind: Pod
metadata:
  name: webserver
spec:
  containers:
  - name: nginx
    image: nginx
    ports:
    - containerPort: 80
EOF
pod/webserver created
$ etcdctl --endpoints=$ETCD_URL \
  --cert /etc/kubernetes/pki/etcd/server.crt \
  --key /etc/kubernetes/pki/etcd/server.key \
  --cacert /etc/kubernetes/pki/etcd/ca.crt \
  get /registry/pods/default/webserver --prefix -w simple
/registry/pods/default/webserver
...
10.244.0.5"
```

> 客户端工具创建资源对象到然后存储到 etcd 的流程

![Minion](https://img-blog.csdnimg.cn/img_convert/1ebad43cb82706dac0885871582c1a14.png)

1. **客户端工具(kubectl) 提供一个期望状态的资源对象的序列化表示，是一样yaml格式提供**
2. **kubectl 将YAML转换为JSON 格式， 并发送给API -server**
3. **对应同类型对象的不同版本，API- SERVER 执行无损转换，对于老版本不存在的字段则存储在annotations中**
4. **API- SERVER 将接收到的对象转换为规范存储版本，这个版本由API-Server启动参数指定，通常是最稳定的版本**
5. **最后将资源对象通过JSON 或YAML方式解析并通过一个特定的key 存入etcd中** 
