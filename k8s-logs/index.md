# K8s Logs


# kubernetes 日志查看工具

## kubetail

[kubetail](https://github.com/johanhaleby/kubetail)

kubetail是一个bash脚本， 可以将多个pod的日志聚合到一个输出流中，并且支持彩色输出和条件过滤
<!--more-->

```bash
 liuhongfeng  kubetail -h
kubetail <search term> [-h] [-c] [-n] [-t] [-l] [-f] [-d] [-P] [-p] [-s] [-b] [-e] [-j] [-k] [-z] [-v] [-r] [-i] -- tail multiple Kubernetes pod logs at the same time

where:
    -h, --help              Show this help text.
    -c, --container         The name of the container to tail in the pod (if multiple containers are defined in the pod).
                            Defaults to all containers in the pod. Can be used multiple times.
    -t, --context           The k8s context. ex. int1-context. Relies on ~/.kube/config for the contexts.
    -l, --selector          Label selector. If used the pod name is ignored.
    -n, --namespace         The Kubernetes namespace where the pods are located. Defaults to "go-zero-baremetal".
    -f, --follow            Specify if the logs should be streamed. (true|false) Defaults to true.
    -d, --dry-run           Print the names of the matched pods and containers, then exit.
    -P, --prefix            Specify if add the pod name prefix before each line. (true|false) Defaults to true.
    -p, --previous          Return logs for the previous instances of the pods, if available. (true|false) Defaults to false.
    -s, --since             Only return logs newer than a relative duration like 5s, 2m, or 3h. Defaults to 10s.
    -b, --line-buffered     This flags indicates to use line-buffered. (true|false) Defaults to false.
    -e, --regex             The type of name matching to use (regex|substring). Defaults to substring.
    -j, --jq                If your output is json - use this jq-selector to parse it. Defaults to "".
                            example: --jq ".logger + \" \" + .message"
    -k, --colored-output    Use colored output (pod|line|false).
                            pod = only color pod name, line = color entire line, false = don't use any colors.
                            Defaults to line.
    -z, --skip-colors       Comma-separated list of colors to not use in output.
                            If you have green foreground on black, this will skip dark grey and some greens: -z 2,8,10
                            Defaults to: 7,8.
        --timestamps        Show timestamps for each log line. (true|false) Defaults to false.
        --tail              Lines of recent log file to display. Defaults to -1, showing all log lines.
    -v, --version           Prints the kubetail version.
    -r, --cluster           The name of the kubeconfig cluster to use.
    -i, --show-color-index  Show the color index before the pod name prefix that is shown before each log line.
                            Normally only the pod name is added as a prefix before each line, for example "[app-5b7ff6cbcd-bjv8n]",
                            but if "show-color-index" is true then color index is added as well: "[1:app-5b7ff6cbcd-bjv8n]".
                            This is useful if you have color blindness or if you want to know which colors to exclude (see "--skip-colors").
                            Defaults to false.

examples:
    kubetail my-pod-v1
    kubetail my-pod-v1 -c my-container
    kubetail my-pod-v1 -t int1-context -c my-container
    kubetail '(service|consumer|thing)' -e regex
    kubetail -l service=my-service
    kubetail --selector service=my-service --since 10m
    kubetail --tail 1
```

## 安装
Just download the [kubetail](https://raw.githubusercontent.com/johanhaleby/kubetail/master/kubetail) file (or any of the [releases](https://github.com/johanhaleby/kubetail/releases)) and you're good to go.

### Homebrew
```bash
$ brew tap johanhaleby/kubetail && brew install kubetail
```

### Linux
```bash
# download and to go
# https://github.com/johanhaleby/kubetail/releases
$ wget https://raw.githubusercontent.com/johanhaleby/kubetail/master/kubetail
$ chmod +x kubetail
$ cp kubetail /usr/local/bi
```

### zsh plugin
```bash
# oh-my-zsh
$ cd ~/.oh-my-zsh/custom/plugins/
$ git clone https://github.com/johanhaleby/kubetail.git kubetail

$ vim ~/.zshrc
plugins=( ... kubetail )

$ source ~/.zshrc
```

## 使用

**展示命名空间下的pod**
```bash
# show all your pods
$ kubectl get pods -n test
NAME                   READY     STATUS    RESTARTS   AGE
app1-v1-aba8y          1/1       Running   0          1d
app1-v1-gc4st          1/1       Running   0          1d
app1-v1-m8acl          1/1       Running   0          6d
app1-v1-s20d0          1/1       Running   0          1d
app2-v31-9pbpn         1/1       Running   0          1d
app2-v31-q74wg         1/1       Running   0          1d
my-demo-v5-0fa8o       1/1       Running   0          3h
my-demo-v5-yhren       1/1       Running   0          2
```

**一次追踪两个“app2”的日志**
```bash
$ kubetail app2
```

**仅追踪多个pod的特定容器的，指定容器**
```bash
$ kubetail app2 -c container1
```

**使用-c 追踪多个特定容器**
```bash
$ kubetail app2 -c container1 -c container2
```

**同时追随多个pod，使用逗号分隔**
```bash
$ kubetail app1,app2
```

**对于高级匹配，您可以使用正则表达式：**
```bash
$ kubetail "^app1|.*my-demo.*" --regex
```

**要尾随特定命名空间内的日志，请确保在为容器和应用程序提供值后附加命名空间标志：**
```bash
$ kubetail app2 -c container1 -n namespace1
```

**使用-k参数，指定kubetail如何使用颜色**
```bash
$ kubetail app2 -k pod
# pod： 只有pod名称着色且其他输出均使用终端默认颜色
$ kubetail app2 -k line
# line： 整行是彩色的(默认)
$ kubetail app2 -k false
# false: 所有输出都不着色
```

# starn 工具

> kubernetes 多pod和container日志追踪
Stern 是使用 Go 语言开发的一款开箱即用的简单工具，它可以将多个 Pod 中的日志信息聚合到一起进行展示，并支持彩色输出和条件过滤

## 工具安装
Homebrew
```bash
$ brew install stern
```
Linux
```bash
# download and to go
# https://github.com/wercker/stern/tags
$ wget https://github.com/wercker/stern/releases/download/1.11.0/stern_linux_amd64
$ chmod +x stern_linux_amd64
$ mv stern_linux_amd64 /usr/local/bin
```

zsh plugin
```bash
# bash-completion
$ brew install bash-completion
$ source <(brew --prefix)/etc/bash-completion
$ source <(stern --completion=bash)

# .zshrc
$ source <(stern --completion=zsh)
```

## 介绍了工具的使用方式
```bash
# 查看默认名称空间下的所有Pod日志
$ stern  .

# 查看 Pod 中指定容器的日志
$ stern app2 --container container1

# 查看指定命名空间中容器的日志
$ stern app2 --namespace namespace1

# 查看指定命名空间中除指定容器外的所有容器的日志
$ stern --namespace namespace1 --exclude-container container1 .

# 查看指定时间范围内容器的日志(15分钟内)
$ stern app2 -t --since 15m

# 查看所有命名空间中符合指定标签容器的日志
$ stern --all-namespaces -l run=nginx

# 查找前端Pod中版本为canary的日志
$ stern frontend --selector release=canary

# 将日志消息通过管道传输到jq命令
$ stern backend -o json | jq .

# 仅输出日志消息本身
$ stern backend -o raw

# 使用自定义模板输出
$ stern --template '{{.Message}} ({{.Namespace}}/{{.PodName}}/{{.ContainerName}})' backend

# 使用stern提供的颜色的自定义模板输出
$ stern --template '{{.Message}} ({{.Namespace}}/{{color .PodColor .PodName}}/{{color .ContainerColor .ContainerName}})' backend
```


