# Kuberenetes SharedMemory


# 什么是 Shared Memory

共享内存是 `Unix` 下的多进程之间的通信方式之一，它使得多进程可以访问同一块内存空间， 不通进程可以及时看到对方进程中共享内存中数据的更新

由于多个进程共享一段内存，这种方式需要依靠某种同步操作， 如互斥锁和信号量等来达到进程间的同步及互斥， 可以说 `Shared Memory` 是最有用的进程间通信方式。

> 以上对 Shared Memory 的解说摘自 进`程间通信 IPC (InterProcess Communication)` 这篇文章

## Docker

查看docker 容器内存的共享内存
```bash
➜ docker run -it --rm  lqshow/busybox-curl:1.28 sh
/data # df -h /dev/shm
Filesystem                Size      Used Available Use% Mounted on
shm                      64.0M         0     64.0M   0% /dev/shm
```

Docker 默认分配的共享内存大小就是 64MB

可以通过设置参数 `--shm-size` 来调整默认的共享内存大小
```bash
docker run -it --rm --shm-size 256M lqshow/busybox-curl:1.28 sh
/data # df -h /dev/shm
Filesystem                Size      Used Available Use% Mounted on
shm                     256.0M         0    256.0M   0% /dev/shm
```

## Kubernetes

k8s 内 Pod 內的实际运行情况
```bash
cat <<EOF | kubectl create -f -
apiVersion: v1
kind: Pod
metadata:
  name: hello-world
  labels:
    name: hello-world
spec:
  containers:
  - image: lqshow/busybox-curl:1.28
    name: hello-world
    command: ['sh', '-c']
    args:
      - while true; do
          echo hello-world;
          sleep 1;
        done
EOF
```

我们在`Kubernetes` 创建的 `Pod`内， 发现其共享内存默认也是 `64MB`， `Docker` 容器內默认的共享内存大小是一样的。
```bash
kubectl exec -it hello-world -- df -h /dev/shm

Filesystem                Size      Used Available Use% Mounted on
shm                      64.0M         0     64.0M   0% /dev/shm
```

我们尝试用 dd 命令去写 100M 的内容， 系统无法完整的写入，提示` No space left on device` 错误
```bash
kubectl exec -it hello-world sh

/data # dd if=/dev/zero of=/dev/shm/output bs=1M count=100
dd: writing '/dev/shm/output': No space left on device
65+0 records in
64+0 records out
67108864 bytes (64.0MB) copied, 0.060007 seconds, 1.0GB/s
```

再次做下确认，发现共享内存其实已经被 dd 写满 64MB 了。

```bash
/data # df -h /dev/shm
Filesystem                Size      Used Available Use% Mounted on
shm                      64.0M     64.0M         0 100% /dev/shm
```

pod 里缺失无法使用超过 `64MB` 的 `Shared Memory`， 

## 解决方案

`Kubernetes` 的资源模型中， 可以将 `pod` 中 `Memory` 资源在 `limits` 中做配置（ `spec.containers[].resources.limits.memory` ），但是它只对`Cgroups `中的 `memory.limit_in_bytes` 起作用，并不会作用到 `Shared Memory `中。

> 官方文档有提到，可以通过将 emptyDir 挂载到 /dev/shm，并将介质类型设置为Memory 来解决这个问题。


## 1、 只设置介质类型

脚本放在 Kubernetes 集群內执行

```bash
cat <<EOF | kubectl create -f -
apiVersion: v1
kind: Pod
metadata:
  name: hello-world
  labels:
    name: hello-world
spec:
  volumes:
  - name: dshm
    emptyDir:
      # 只设置介质类型
      medium: Memory
  containers:
  - image: lqshow/busybox-curl:1.28
    name: hello-world
    command: ['sh', '-c']
    args:
      - while true; do
          echo hello-world;
          sleep 1;
        done
    volumeMounts:
    - mountPath: /dev/shm
      name: dshm
EOF
```

查看 `pod` 的情况
```bash
kubectl exec -it hello-world -c hello-world -- df -h /dev/shm

Filesystem                Size      Used Available Use% Mounted on
tmpfs                    14.7G         0     14.7G   0% /dev/shm
```

查看 `pod` 所在节点的内存情况
```bash
ssh 138 'free -h'
              total        used        free      shared  buff/cache   available
Mem:            29G         23G        2.6G        1.4G        3.6G         20G
Swap:            0B          0B          0B
```

虽然未设置 `sizeLimit`，但是显示分配了 `HOST` 主机上将近一半的内存。

`HOST` 主机上提示 可用的内存其实只有 `2.6G` 了，但是 Pod 內提示`Available` 的还有 `14.7G`，

## 2、 设置介质类型， 同时配置 sizeLimit

`Deployment` 资源， 
```bash
cat <<EOF | kubectl create -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello-world
  labels:
    app: hello-world
spec:
  replicas: 1
  selector:
    matchLabels:
      app: hello-world
  template:
    metadata:
      labels:
        app: hello-world
    spec:
      volumes:
      - name: dshm
        emptyDir:
          # 设置介质类型，且配置 sizeLimit
          medium: Memory
          sizeLimit: 256Mi
      containers:
      - name: hello-world
        image: lqshow/busybox-curl:1.28
        command: ['sh', '-c']
        args:
        - while true; do
            echo hello-world;
            sleep 1;
          done
        volumeMounts:
        - mountPath: /dev/shm
          name: dshm
EOF
```

我们通过实际运行的情况，发现即便设置了 sizelimt 限制，但是实际显示仍然分配了 HOST 主机上将近一半的内存，和第一个测试结果是一模一样的
```bash
kubectl exec -it $(kubectl get pod -l app=hello-world --no-headers|grep -v "Evicted"|awk '{print $1}') -c hello-world -- df -h /dev/shm

Filesystem                Size      Used Available Use% Mounted on
tmpfs                    14.7G         0     14.7G   0% /dev/shm
```

使用`dd` 尝试去做验证
```bash
kubectl exec -it $(kubectl get pod -l app=hello-world --no-headers|grep -v "Evicted"|awk '{print $1}') -c hello-world sh

# 1. 先尝试直接写满 256 M
/data # dd if=/dev/zero of=/dev/shm/output bs=1M count=256
256+0 records in
256+0 records out
268435456 bytes (256.0MB) copied, 0.520999 seconds, 491.4MB/s
/data #
/data # df -h /dev/shm
Filesystem                Size      Used Available Use% Mounted on
tmpfs                    14.7G    256.0M     14.4G   2% /dev/shm
/data #

# 2. 再尝试写入 100 M，并没有看到出现 `No space left on device` 的提示
/data # dd if=/dev/zero of=/dev/shm/output2 bs=1M count=100
100+0 records in
100+0 records out
104857600 bytes (100.0MB) copied, 0.227946 seconds, 438.7MB/s
/data #

# 3. 发现成功的写入了 100 M
/data # df -h /dev/shm
Filesystem                Size      Used Available Use% Mounted on
tmpfs                    14.7G    356.0M     14.4G   2% /dev/shm

# 4. 在一定时间间隔內，Pod 会因为异常被驱逐
/data # command terminated with exit code 137
```

`sizeLimit` 实际上是起了作用的，`Pod` 最后是以 137 CODE 退出了

被驱逐 Pod 的 Event 信息，只保留了重要的提示
```bash
...
Events:
  Type     Reason     Age        From               Message
  ----     ------     ----       ----               -------
  ...
  Warning  Evicted    20m        kubelet, kind-dev4  Usage of EmptyDir volume "dshm" exceeds the limit "256Mi".
  ...
```

> 当宿主机资源紧张的情况下，kubelet 会主动地结束 Pod 以回收短缺的资源。具体驱逐 Pod

## 升级 Kubernetes 集群版本
从官方文档给出的提示可以看出，SizeMemoryBackedVolumes 这个特性在我们的集群內并没有生

SizeMemoryBackedVolumes 是从 1.20 版本才开始支持


