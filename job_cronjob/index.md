# Kubernets-Job_Cronjob


# Job 与 CronJob

### Job API

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: pi
spec:
  template:
    spec:
      containers:
      - name: pi
        image: resouer/ubuntu-bc 
        command: ["sh", "-c", "echo 'scale=10000; 4*a(1)' | bc -l "]
      restartPolicy: Never
  backoffLimit: 4
```

计算π值的容器。而通过 scale=10000，我指定了输出的小数点后的位数是 10000

### 创建job

```yaml
$ kubectl create -f job.yaml
```

查看job 对象

```bash
$ kubectl describe jobs/pi
Name:             pi
Namespace:        default
Selector:         controller-uid=c2db599a-2c9d-11e6-b324-0209dc45a495
Labels:           controller-uid=c2db599a-2c9d-11e6-b324-0209dc45a495
                  job-name=pi
Annotations:      <none>
Parallelism:      1
Completions:      1
..
Pods Statuses:    0 Running / 1 Succeeded / 0 Failed
Pod Template:
  Labels:       controller-uid=c2db599a-2c9d-11e6-b324-0209dc45a495
                job-name=pi
  Containers:
   ...
  Volumes:              <none>
Events:
  FirstSeen    LastSeen    Count    From            SubobjectPath    Type        Reason            Message
  ---------    --------    -----    ----            -------------    --------    ------            -------
  1m           1m          1        {job-controller }                Normal      SuccessfulCreate  Created pod: pi-rq5rl
```

pod模板， 会自动加上一个controller-uid= 随机字符串

job对象本身， 也会自动加上label的对应的Selector， 保证job与他所管理的Pod之间的匹配关系

这种自动生成的Label对用户来说并不友好， 所以不太适合推广到Deployment等长作业编排对象上。

> 事实上， restartPolicy在job对象里只允许被设置为Never 和 OnFailure
而在deployment对象里， restartPolicy则只允许被设置为Always
> 

### 查看pod日志

```bash
$ kubectl logs pi-rq5rl
3.141592653589793238462643383279...
```

### 离线作业失败了怎么办？

**job 定义了 restartPolicy=Never， 那么离线作业失败后 job Controller 就会不断尝试创建一个新的pod**

在job对象的spec.backoffLimit字段字段里定义了重试次数为 4（即，backoffLimit=4）， 默认为6

重新创建Pod的间隔是呈指数增加的， 重新创建Pod的发生在 10 s、20 s、40 s

定义 restartPolicy=OnFailure, 那么离线作业失败后， JOb Controller就不会尝试创建新的Pod， 但是会不断尝试重启Pod的容器

在 spec.activeDeadlineSeconds 字段可以设置最长运行时间

```bash
spec:
 backoffLimit: 5
 activeDeadlineSeconds: 100
```

一旦运行了100s, 这个Job的所有Pod都会被终止

你可以在 Pod 的状态里看到终止的原因是 reason: DeadlineExceeded。

## Job Controller 对并行作业的控制方法

负责控制并行控制的两个参数

- spec.parallelism:  定义一个Job在任意时间最多可以启动多少个Pod同事运行
- spec.completions: 定义Job至少要完成的Pod数目， 即Job的最小完成数

创建一个参数例子

```bash
apiVersion: batch/v1
kind: Job
metadata:
  name: pi
spec:
  parallelism: 2
  completions: 4
  template:
    spec:
      containers:
      - name: pi
        image: resouer/ubuntu-bc
        command: ["sh", "-c", "echo 'scale=5000; 4*a(1)' | bc -l "]
      restartPolicy: Never
  backoffLimit: 4
```

创建job

```bash
$ kubectl create -f job.yaml
```

查看job

```bash
$ kubectl get job
NAME      DESIRED   SUCCESSFUL   AGE
pi        4         0            3s
```

DESIRED的值 正是completions定义的最小完成数

```bash
$ kubectl get pods
NAME       READY     STATUS    RESTARTS   AGE
pi-gmcq5   0/1       Completed   0         40s
pi-84ww8   0/1       Pending   0         0s
pi-5mt88   0/1       Completed   0         41s
pi-62rbt   0/1       Pending   0         0s
 
$ kubectl get pods
NAME       READY     STATUS    RESTARTS   AGE
pi-gmcq5   0/1       Completed   0         40s
pi-84ww8   0/1       ContainerCreating   0         0s
pi-5mt88   0/1       Completed   0         41s
pi-62rbt   0/1       ContainerCreating   0         0s
```

由于所有Pod均成功退出, job执行完成, 看懂SuCCESSFUL为4

```bash
$ kubectl get pods 
NAME       READY     STATUS      RESTARTS   AGE
pi-5mt88   0/1       Completed   0          5m
pi-62rbt   0/1       Completed   0          4m
pi-84ww8   0/1       Completed   0          4m
pi-gmcq5   0/1       Completed   0          5m
 
$ kubectl get job
NAME      DESIRED   SUCCESSFUL   AGE
pi        4         4            5m
```

### Job Controller的工作原理

- 首先 job Controller的控制对象直接就是 Pod
- 其次, job Controller在控制循环中进行的协调操作,  是根据实际的Running状态Pod额数目,已经成功退出的Pod的数目, 以及 parallelism、 conpletions 参数的值共同计算出在这个周期里，应该创建或者删除的Pod数目，然后调用Kubernetes API执行这个操作

### 第一种： 外部管理器+Job模板

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: process-item-$ITEM
  labels:
    jobgroup: jobexample
spec:
  template:
    metadata:
      name: jobexample
      labels:
        jobgroup: jobexample
    spec:
      containers:
      - name: c
        image: busybox
        command: ["sh", "-c", "echo Processing item $ITEM && sleep 5"]
      restartPolicy: Never
```

控制这种 Job 时，我们只要注意如下两个方面即可：

1. 创建 Job 时，替换掉 $ITEM 这样的变量；
2. 所有来自于同一个模板的 Job，都有一个 jobgroup: jobexample 标签，也就是说这一组 Job 使用这样一个相同的标识。

```yaml
$ mkdir ./jobs
$ for i in apple banana cherry
do
  cat job-tmpl.yaml | sed "s/\$ITEM/$i/" > ./jobs/job-$i.yaml
done
```

```yaml
$ kubectl create -f ./jobs
$ kubectl get pods -l jobgroup=jobexample
NAME                        READY     STATUS      RESTARTS   AGE
process-item-apple-kixwv    0/1       Completed   0          4m
process-item-banana-wrsf7   0/1       Completed   0          4m
process-item-cherry-dnfu9   0/1       Completed   0          4m
```

### 第二种： 拥有固定任务数目的并行Job

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: job-wq-1
spec:
  completions: 8
  parallelism: 2
  template:
    metadata:
      name: job-wq-1
    spec:
      containers:
      - name: c
        image: myrepo/job-wq-1
        env:
        - name: BROKER_URL
          value: amqp://guest:guest@rabbitmq-service:5672
        - name: QUEUE
          value: job1
      restartPolicy: OnFailure
```

### 第三种： 指定并行度（parallelism）， 但不设置固定的completions

这种情况 ， 任务的总数是未知的， 所以不仅需要一个工作队列来负责任务分发， 还需要判断工作列表已经为空。

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: job-wq-2
spec:
  parallelism: 2
  template:
    metadata:
      name: job-wq-2
    spec:
      containers:
      - name: c
        image: gcr.io/myproject/job-wq-2
        env:
        - name: BROKER_URL
          value: amqp://guest:guest@rabbitmq-service:5672
        - name: QUEUE
          value: job2
      restartPolicy: OnFailure
```

## CronJob（定时任务）

CronJob是一个专门用来管理Job对象的控制器， 它的创建和删除job的依据， 是schedule字段的定义一个标准的[Unix Cron](https://en.wikipedia.org/wiki/Cron)格式的表达式。

```yaml
apiVersion: batch/v1beta1
kind: CronJob
metadata:
  name: hello
spec:
  schedule: "*/1 * * * *"
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: hello
            image: busybox
            args:
            - /bin/sh
            - -c
            - date; echo Hello from the Kubernetes cluster
          restartPolicy: OnFailure
```

由于定时任务的特殊性， 某个job还没有执行完， 另一个新的job就产生了， 通过spec.concurrencyPolicy 字段来定义具体的处理策略

1. concurrencyPolicy=Allow， 默认情况， 意味着这些job可以同时存在
2. concurrencyPolicy=Forbid， 意味着不会创建新的pod， 该创建周期被跳过
3. concurrencyPolicy=Replace， 意味着新产生的job会替换旧的、没有执行完的job

如果某一次 Job 创建失败，这次创建就会被标记为“miss”。当在指定的时间窗口内，miss 的数目达到 100 时，那么 CronJob 会停止再创建这个 Job

这个时间窗口，可以由 spec.startingDeadlineSeconds 字段指定。比如 startingDeadlineSeconds=200，意味着在过去 200 s 里，如果 miss 的数目达到了 100 次，那么这个 Job 就不会被创建执行了。
