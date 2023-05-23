# Golang Etcd


# Etcd
* etcd 是一个开源的， 高可用的分布式的 key-value 存储系统， 可以用于配制共享和服务的注册和发现

etcd具有的特点
* 完全复制： 集群中的每个阶段都可以使用完整的文档
* 高可用性： Etcd可用于避免硬件的单点故障或网络问题
* 一致性： 每次读取都会返回跨多主机的最新写入
* 简单： 包括一个定义良好、面向用户的API（gRPC）
* 安全： 实现了带有可选的客户端证书身份验证的自动化TLS
* 快速： 每秒10000次写入的基准速度
* 可靠： 用Raft算法实现了强一致、高可用的服务存储目录

## etcd 的应用场景

> 服务发现
服务发现要解决的也是分布式系统中最常见的问题之一，即在同一个分布式集群中的进程或服务，要如何才能找到对方并建立连接。本质上来说，服务发现就是想要了解集群中是否有进程在监听 udp 或 tcp 端口，并且通过名字就可以查找和连接。

{{< figure src="/images/etcd_Registry.png" title="Registry (figure)" >}}


![](media/16848508417516.png)
> 配置中心
* 将一些配置存放在etcd 上集中管理
* 这类场景的使用方式通常是这样：应用在启动的时候主动从 etcd 获取一次配置信息，同时，在 etcd 节点上注册一个 Watcher 并等待，以后每次配置有更新的时候，etcd 都会实时通知订阅者，以此达到获取最新配置信息的目的

> 分布式锁
* 因为 etcd 使用 Raft 算法保持了数据的强一致性，某次操作存储到集群中的值必然是全局一致的，所以很容易实现分布式锁。锁服务有两种使用方式，一是保持独占，二是控制时序。
    * 保持独占即所有获取锁的用户最终只有一个可以得到。etcd 为此提供了一套实现分布式锁原子操作 CAS（CompareAndSwap）的 API。通过设置prevExist值，可以保证在多个节点同时去创建某个目录时，只有一个成功。而创建成功的用户就可以认为是获得了锁。
    * 控制时序，即所有想要获得锁的用户都会被安排执行，但是获得锁的顺序也是全局唯一的，同时决定了执行顺序。etcd 为此也提供了一套 API（自动创建有序键），对一个目录建值时指定为POST动作，这样 etcd 会自动在目录下生成一个当前最大的值为键，存储这个新的值（客户端编号）。同时还可以使用 API 按顺序列出所有当前目录下的键值。此时这些键的值就是客户端的时序，而这些键中存储的值可以是代表客户端的编号
    

{{< figure src="/images/etcd_lock.png" title="etcd_lock (figure)" >}}

![](media/16848512370817.png)
## go 操作etcd

## 安装
```bash
go get go.etcd.io/etcd/clientv3
```

## put和get 操作
```go
package main

import (
	"context"
	"fmt"
	"time"

	"go.etcd.io/etcd/clientv3"
)

// etcd client put/get demo
// use etcd/clientv3

func main() {
	cli, err := clientv3.New(clientv3.Config{
		Endpoints:   []string{"127.0.0.1:2379"},
		DialTimeout: 5 * time.Second,
	})
	if err != nil {
		// handle error!
		fmt.Printf("connect to etcd failed, err:%v\n", err)
		return
	}
    fmt.Println("connect to etcd success")
	defer cli.Close()
	// put
	ctx, cancel := context.WithTimeout(context.Background(), time.Second)
	_, err = cli.Put(ctx, "demo", "dsb")
	cancel()
	if err != nil {
		fmt.Printf("put to etcd failed, err:%v\n", err)
		return
	}
	// get
	ctx, cancel = context.WithTimeout(context.Background(), time.Second)
	resp, err := cli.Get(ctx, "demo")
	cancel()
	if err != nil {
		fmt.Printf("get from etcd failed, err:%v\n", err)
		return
	}
	for _, ev := range resp.Kvs {
		fmt.Printf("%s:%s\n", ev.Key, ev.Value)
	}
}
```
