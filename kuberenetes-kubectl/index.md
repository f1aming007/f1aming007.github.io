# Kuberenetes Kubectl

# kubernetes 常用运维 kubectl命令

## 环境上下文切换

### 切换集群
```bash

# 获取所有的 contexts
kubectl config get-contexts

# 切换到指定的 context
kubectl config use-context <CONTEXT-NAME>
```

### 设置默认的namespace
```bash

# 将当前上下文设置为指定的 namespace
kubectl config set-context --current --namespace=<MY_NAMESPACE>

# 验证当前上下文是否是正确设置的 namespace
kubectl config view --minify | grep namespace
```

> 平时用的最顺手的，还数 `kubectx + kubens` 的组合，但是大部分客户的环境并没有预装该工具，还是需要记录下原生命令
> [kubectx](https://github.com/ahmetb/kubectx)

## 列出 non-running Pods

当前集群内有哪些pod处于异常状态， 帮助运维工程师快速定位系统是否有异常
```bash
kubectl get pods -A --no-headers | grep -v Running | grep -v Completed
```

## 资源排序
以 pod资源作为例子， 提供常用的排序方式

```bash
# 根据启动时间降序（descending order）
kubectl get pods --sort-by=.metadata.creationTimestamp
```

```bash
# 根据启动时间升序（ascending order）
kubectl get pods --sort-by=.metadata.creationTimestamp | awk 'NR == 1; NR > 1 {print $0 | "tac"}'
kubectl get pods --sort-by=.metadata.creationTimestamp | tail -n +2 | tac
kubectl get pods --sort-by={metadata.creationTimestamp} --no-headers | tac
kubectl get pods --sort-by=.metadata.creationTimestamp | tail -n +2 | tail -r
```

```bash
# 根据重启次数排序
kubectl get pods -A --sort-by='.status.containerStatuses[0].restartCount'
```

## 导出干净的 YAML
```bash
kubectl get cm nginx-config -o yaml | kubectl neat -o yaml
```

## 释放集群内资源
临时将某个namespace下的Pod关闭

如果你需要释放某个 namespace 下的资源，但是又不想删除 Kubernetes 集群內的信息，可通过调整  replicas 来解决
```bash
# 方法一：通过 patch 模式
kubectl get deploy -o name -n <NAMESPACE>|xargs -I{} kubectl patch {} -p '{"spec":{"replicas":0}}'

# 方法二：通过资源伸缩副本数
kubectl get deploy -o name |xargs -I{} kubectl scale --replicas=0 {}
```

### 临时关闭 Daemonsets
如果需要临时将 Daemonsets 关闭，只需要将其调度到一个不存在的 node 上即可，调整下 nodeSelector
```bash
kubectl patch daemonsets nginx-ingress-controller -p '{"spec":{"template":{"spec":{"nodeSelector":{"project/xdp":"none"}}}}}'
```

## 批量删除某种状态下资源
批量强制删除 当前 namespace 下所有处于 Terminating 状态的 Pod 资源
```bash
kubectl get pod |grep Terminating|awk '{print $1}'|xargs kubectl delete pod --grace-period=0 --force
```

批量强制删除 集群內所有 namespace 下所有处于 Terminating 状态的 Pod 资源
```bash
for ns in $(kubectl get ns --no-headers | cut -d ' ' -f1); do \
  for po in $(kubectl -n $ns get po --no-headers --ignore-not-found | grep Terminating | cut -d ' ' -f1); do \
    kubectl -n $ns delete po $po --force --grace-period 0; \
  done; \
done;
```

## 资源复制
如果当前 Namespace 下的某个资源，其他 Namespace 下也需要，可参考以下方法，以复制 secret 举例。
```bash
kubectl get secret <SECRET-NAME> -n <SOURCE-NAMESPACE> -oyaml | sed "/namespace:/d" | kubectl apply --namespace=<TARGET-NAMESPACE> -f -
```

## 使用交互 shell 访问匹配到 标签的 Pod
```bash

# example 1
kubectl exec -i -t $(kubectl get pod -l <KEY>=<VALUE> -o name |sed 's/pods\///') -- bash

# example 2
kubectl exec -i -t $(kubectl get pod -l <KEY>=<VALUE> -o jsonpath='{.items[0].metadata.name}') -- bash
```

## 在多个 Pod 中运行命令
```bash
kubectl get pods -o name | xargs -I{} kubectl exec {} -- <command goes here>
```

## 查看所有镜像
```bash
kubectl get pods -o custom-columns='NAME:metadata.name,IMAGES:spec.containers[*].image'
```

## 设置环境变量
```bash
kubectl set env deploy <DEPLOYMENT_NAME> OC_XXX_HOST=bbb
```

## 充重置集群节点
```bash
# 1. 将节点标记为不可调度，确保新的容器不会调度到该节点
kubectl cordon <NODE-NAME>

# 2. Master 节点上将需要重置的节点驱逐, 除了 deemonset
kubectl drain <NODE-NAME> --delete-local-data --force --ignore-daemonsets

# 3. 删除节点
kubectl delete node <NODE-NAME>

# 4. 在需要重置节点上执⾏重置脚本，注意，如果在 Master 主节点执⾏ kubeadm reset，则需要重新初始化集群
kubeadm reset
```

## Forward local port to a pod or service
为 pod 创建本地端口映射
```bash
# 将 localhost:3000 的请求转发到 nginx-pod Pod 的 80 端口
kubectl port-forward nginx-po 3000:80
```
为 service 创建本地端口映射
```bash
# 将 localhost:3201 的请求转发到 nginx-web service 的 3201 端口
kubectl port-forward svc/nginx-web 3201
```
> 一个端口转发的工具[kubewd](https://github.com/txn2/kubefwd)

## 创建一个临时可调试的Pod
退出 shell，可自动删除 Pod
```bash
kubectl run ephemeral-busybox \
  --rm \
  --stdin \
  --tty \
  --restart=Never \
  --image=lqshow/busybox-curl:1.28 \
  -- sh
```

## 故障排除

```bash

# 1. 查看资源详情
kubectl describe
# 2. 查看 Pod 内容器的日志
kubectl logs
# 3. 在指定容器内执行命令
kubectl exec
# 4. 查看节点列表明细
kubectl get nodes --show-labels
# 5. 查看所有事件
kubectl get events
```

参考
https://kubernetes.io/zh/docs/reference/kubectl/cheatsheet/

https://ahmet.im/blog/mastering-kubeconfig/
