# K8s Liveness_Readines

# Liveness&Readines
![Minion](https://plus.unsplash.com/premium_photo-1675404521308-2ceb136f3908?ixlib=rb-4.0.3&ixid=M3wxMjA3fDB8MHxwaG90by1wYWdlfHx8fGVufDB8fHx8fA%3D%3D&auto=format&fit=crop&w=1932&q=80)
## 需求来源
1. 实时观察应用的健康状态
2. 获取应用的资源使用情况
3. 拿到应用的实时日志， 进行问题的诊断和分析
<!--more-->
## Liveness probe 和 Readiness probe 介绍

* livenness probe 是就绪指针，判断一个pod是否处于就绪状态。
* 当一个pod处于就绪状态的时候 才能对外提供对应的服务，接入层的流量才能打入相应的pod
* 当这个pod不处于就绪状态， 接入层灰吧相应的流量从这个pod上面剔除。
{{< figure src="/images/Readiness01.png" title="Liveness01 (figure)" >}}
 
 * 这个pod指针一直处于失败， 接入层流量不会打到这个pod
 
* 这个 pod 的状态从 FAIL 的状态转换成 success 的状态时，它才能够真实地承载这个流量
* 这个时候会由上层的判断机制来判断这个 pod 是否需要被重新拉起。那如果上层配置的重启策略是 restart always 的话，那么此时这个 pod 会直接被重新拉起。

## 应用健康状态-使用方式

### 探测方式
liveness指针和Readkiness指针支持三种不同的探测方式
1. httpGet: 通过发送http 个体请求进行判断， 当返回码是200～399 ，标识这个应用是健康的
2. Exec: 执行容器中的一个命令判断当前容器是否正常，当返回时0，标识容器时健康的
3. tcpSocket： 通过探测容器的IP和Port 进行TCP健康检查， TCP能正常建立， 标识这个容器时健康的

### 探测结果
1. Success:  标识Container 通过健康检查
2. Failure:  container没有通过健康检查，此时会有一个相应的处理，Readiness——（service层将没有通过Readiness的pod进行移除）liveness 将这个pod重新拉起， 或者删除
3. Unknown: 当前的执行机制没有完成执行（超时或者一些脚本没有及时返回），等待下次的机制来进行校验

## 应用健康状态检查方式-pod Probe spec

### exec
![Minion](https://intranetproxy.alipay.com/skylark/lark/0/2019/png/225439/1570502713702-6b90ff5b-265d-4c2d-a46f-b363ea54c2d7.png)

* 通过cat 一个具体文件来判断当前Liveness probe 的状态，返回0 这个pod 处于健康状态
![Minion](https://intranetproxy.alipay.com/skylark/lark/0/2019/png/225439/1570502757691-ece72c57-6446-4796-8e4b-568a1cf6ca74.png)

### httpGet


### tcpSocket
![Minion](https://intranetproxy.alipay.com/skylark/lark/0/2019/png/225439/1570502777983-dfa814ec-747e-4540-9f6a-46e41912f9bd.png)
* 参数说明：
    * initialDelaySeconds： pod启动延迟多久进行一次检查
    * periodSeconds： 检查的时间间隔，默认值是10秒
    * timeoutSeconds： 检查的超时时间
    * successThreshold： pod从探测失败到再一次探测成功， 锁需要的阀值次数
    * failureThreshold： 探测失败的重启次数， 默认值是3

## Liveness 与 Readiness 总结
![Minion](https://intranetproxy.alipay.com/skylark/lark/0/2019/png/225439/1570502867094-9c716cdd-edd0-4296-98c1-02afab9a5487.png?x-oss-process=image/resize,w_2222)

* liveness: 判断容器是否处于running ,如果容器不健康，会通过kubelet 杀掉相应的pod
* Readiness： 判断容器是否启动完成，如果探测一个结果不成功， 会从pod上Endpoint上移除。

### 适用场景
* liveness： 支持可以重新拉起的应用
* Readiness： 应对 启动后无法立即对外提供服务的应用。


## 应用故障排除——状态机制

* k8s 的状态机制， 通过yaml的方式定义个期望到达的状态。
* 这个yaml的真正执行过程 各种各样的controller来负责整体的状态之间的转换
![Minion](https://intranetproxy.alipay.com/skylark/lark/0/2019/png/225439/1570502911373-048d928b-f606-4503-ac09-ab71505841cf.png)
![Minion](https://intranetproxy.alipay.com/skylark/lark/0/2019/png/225439/1570502921155-cb806fc8-e90d-4c81-bd1f-8636aa59db66.png)

* 一个 Pod 的一个生命周期。刚开始它处在一个 pending 的状态，那接下来可能会转换到类似像 running，也可能转换到 Unknown，甚至可以转换到 failed。然后，当 running 执行了一段时间之后，它可以转换到类似像 successded 或者是 failed，然后当出现在 unknown 这个状态时，可能由于一些状态的恢复，它会重新恢复到 running 或者 successded 或者是 failed。
* k8s 整体的一个状态就是状态机制的转换
* 一个pod状态位的展现
![Minion](https://intranetproxy.alipay.com/skylark/lark/0/2019/png/225439/1570503040532-79a6ac60-4106-4f37-87b2-9934856626ca.png)

## 应用故障排除-场景应用异常

### pod停留在Pending
* 标识没有调度器进行介入
    * 可能是资源或端口占用
    * 由于node selector 造成的pod无法调度
    
### Pod 停留在 waiting
* 表示pod的镜像没有正常拉取
* 可能原因
    * 私有镜像 没有配置pod secret
    * 镜像地址不存在

### Pod 不断被拉取并且可以看到 crashing
* 标识pod已经调度完成， 但是启动失败
* 需要关注应用的自身状态
    * 查看pod的具体日志

### Pod 处在 Runing 但是没有正常工作
* 常见问题——yaml信息里可能有拼写错误
* apply-validate -f pod.yaml 判断当前的yaml是否正确
    * 如果没有问题，诊断配置的端口是否正常，以及liveness或Readiness 配置是否正确

### Service 无法正常的工作
* 因为service和底层pod之间的关联是通过selector的方式进行匹配的，
* pod上面配置label， 然后service通过match label的方式和pod进行关联
    * 如果lable配置有问题，可能会造成service无法找到后面的endpoint
    * 造成相应的service无法对外提供服务
* 排除这种问题 
    * 首先 看service 后面的是不是有一个真正的endpoint
    * 其次 看这个endpoint 是否可以对外提供正常的服务

## 应用远程调试

### 应用远程调试 - Pod 远程调试

* 进入容器里进行诊断
![Minion](https://intranetproxy.alipay.com/skylark/lark/0/2019/png/225439/1570503283791-1688c1e8-032d-4e35-9be0-728101b6663d.png?x-oss-process=image/resize,w_2222)


### 应用远程调试 - Service 远程调试
* service调试 分了两部分
    1. 将服务暴露到远程的一个集群之内，让远程集群的一些应用去调用本地的一个服务，这是一条反向的一个链路
    2. 让本地的服务能够远程调用服务， 这是一条正向的链路

![Minion](https://intranetproxy.alipay.com/skylark/lark/0/2019/png/225439/1570503307524-cc111087-301f-4db3-9f29-b8f94345699a.png)
* 将 Telepresence 的一个 Proxy 应用部署到远程的 K8s 集群里面。然后将远程单一个 deployment swap 到本地的一个 application，使用的命令就是 Telepresence-swap-deployment 然后以及远程的 DEPLOYMENT_NAME。通过这种方式就可以将本地一个 application 代理到远程的 service 之上、可以将应用在远程集群里面进行本地调试

![Minion](https://intranetproxy.alipay.com/skylark/lark/0/2019/png/225439/1570503322949-7064b86b-cdea-4b70-af98-bbf26e22e27d.png)
* 使用方式是 kubectl port-forward，然后 service 加上远程的 service name，再加上相应的 namespace，后面还可以加上一些额外的参数，比如说端口的一个映射，通过这种机制就可以把远程的一个应用代理到本地的端口之上，此时通过访问本地端口就可以访问远程的服务。

## kubectl-debug
* kubectl-debug 这个工具是依赖于 Linux namespace 的方式来去做的，它可以 datash 一个 Linux namespace 到一个额外的 container，然后在这个 container 里面执行任何的 debug 动作，其实和直接去 debug 这个 Linux namespace 是一致的。这里有一个简单的操作

* 通过 kubectl-debug 这条命令来去诊断远程的一个 pod
    * 执行debug的时候，首先会拉取一些镜像， 这个镜像里会默认带一些诊断工具
    * 当这个镜像启用的时候，它会把这个 debug container 进行启动
    * 这个 container 和相应的你要诊断的这个 container 的 namespace 进行挂靠，类似像网络站，或者是类似像内核的一些参数，可以在这个 debug container 里面实时地进行查看。


![Minion](https://intranetproxy.alipay.com/skylark/lark/0/2019/png/225439/1570503431005-d0a56ed2-4a7d-4a71-bdee-65ce5635d829.png)

* 去查看类似像 hostname、进程、netstat 等等，这个容器和需要debug的pod在同一个环境里
![Minion](https://intranetproxy.alipay.com/skylark/lark/0/2019/png/225439/1570503480829-fe86c647-228f-486e-9f24-edb3a0e202d5.png)

* 进行 logout 的话，相当于会把相应的这个 debug pod 杀掉
![Minion](https://intranetproxy.alipay.com/skylark/lark/0/2019/png/225439/1570503514061-af6ea2ab-4539-41ef-bfd9-530e2ac3a5fd.png)

