# Golang Redis


# Redis

## Redis 介绍
* Redis是一个开源的内存数据库，提供了多种不同的数据结构， 通过复制、持久化/客户端分片等特性， 方便的将Redis 拓展成一个包含数百GB的数据， 处理上百万次请求


> Redis 支持的数据结构
* string
* hashe
* list
* set
* sortedset
* bitmap
* stream

> Redis 应用场景
* 缓存系统，减轻mysql的压力
* 计数系统，
* 热门排行榜
* 利用LIST 实现队列功能
* 利用HyperLog 统计UV、PV等数据
* 使用geospatial index 进行地理位置相关的查询

## docker 安装Reids
* 启动redis
```bash
docker run --name redis507 -p 6379:6379 -d redis:5.0.7
```

* 连接redis server
```bash
docker run -it --network host --rm redis:5.0.7 redis-cli
```

## go-redis库
> 安装
```bash
go get github.com/go-redis/redis/v8
```

> 连接

### 普通模式连接
* 使用redis.NewClient 函数连接Redis服务器
```go
rdb := redis.NewClient(&redis.Options{
	Addr:     "localhost:6379",
	Password: "", // 密码
	DB:       0,  // 数据库
	PoolSize: 20, // 连接池大小
})
```

* 谁也redis.ParseURL 函数表示数据源的字符串解析Redis
```go
opt, err := redis.ParseURL("redis://<user>:<pass>@localhost:6379/<db>")
if err != nil {
	panic(err)
}

rdb := redis.NewClient(opt)
```

* TLS 模式
```go
rdb := redis.NewClient(&redis.Options{
	TLSConfig: &tls.Config{
		MinVersion: tls.VersionTLS12,
		// Certificates: []tls.Certificate{cert},
    // ServerName: "your.domain.com",
	},
})
```

* Redis Sentinel模式
```go
rdb := redis.NewFailoverClient(&redis.FailoverOptions{
    MasterName:    "master-name",
    SentinelAddrs: []string{":9126", ":9127", ":9128"},
})
```

* Redis Cluster模式
```go
rdb := redis.NewClusterClient(&redis.ClusterOptions{
    Addrs: []string{":7000", ":7001", ":7002", ":7003", ":7004", ":7005"},

    // 若要根据延迟或随机路由命令，请启用以下命令之一
    // RouteByLatency: true,
    // RouteRandomly: true,
})
```

## 基本使用
* redis基础操作
```go
// doCommand go-redis基本使用示例
func doCommand() {
	ctx, cancel := context.WithTimeout(context.Background(), 500*time.Millisecond)
	defer cancel()

	// 执行命令获取结果
	val, err := rdb.Get(ctx, "key").Result()
	fmt.Println(val, err)

	// 先获取到命令对象
	cmder := rdb.Get(ctx, "key")
	fmt.Println(cmder.Val()) // 获取值
	fmt.Println(cmder.Err()) // 获取错误

	// 直接执行命令获取错误
	err = rdb.Set(ctx, "key", 10, time.Hour).Err()

	// 直接执行命令获取值
	value := rdb.Get(ctx, "key").Val()
	fmt.Println(value)
}
```

>  执行任意命令
* Do方法可以形象任意命令
```go
// doDemo rdb.Do 方法使用示例
func doDemo() {
	ctx, cancel := context.WithTimeout(context.Background(), 500*time.Millisecond)
	defer cancel()

	// 直接执行命令获取错误
	err := rdb.Do(ctx, "set", "key", 10, "EX", 3600).Err()
	fmt.Println(err)

	// 执行命令获取结果
	val, err := rdb.Do(ctx, "get", "key").Result()
	fmt.Println(val, err)
}
```

>  Redis.Nil
* Nil错误来表示Key不存在的错误
```go
// getValueFromRedis redis.Nil判断
func getValueFromRedis(key, defaultValue string) (string, error) {
	ctx, cancel := context.WithTimeout(context.Background(), 500*time.Millisecond)
	defer cancel()

	val, err := rdb.Get(ctx, key).Result()
	if err != nil {
		// 如果返回的错误是key不存在
		if errors.Is(err, redis.Nil) {
			return defaultValue, nil
		}
		// 出其他错了
		return "", err
	}
	return val, nil
}
```

## zset示例
* go-redis 操作zset
```go
// zsetDemo 操作zset示例
func zsetDemo() {
	// key
	zsetKey := "language_rank"
	// value
	languages := []*redis.Z{
		{Score: 90.0, Member: "Golang"},
		{Score: 98.0, Member: "Java"},
		{Score: 95.0, Member: "Python"},
		{Score: 97.0, Member: "JavaScript"},
		{Score: 99.0, Member: "C/C++"},
	}
	ctx, cancel := context.WithTimeout(context.Background(), 500*time.Millisecond)
	defer cancel()

	// ZADD
	err := rdb.ZAdd(ctx, zsetKey, languages...).Err()
	if err != nil {
		fmt.Printf("zadd failed, err:%v\n", err)
		return
	}
	fmt.Println("zadd success")

	// 把Golang的分数加10
	newScore, err := rdb.ZIncrBy(ctx, zsetKey, 10.0, "Golang").Result()
	if err != nil {
		fmt.Printf("zincrby failed, err:%v\n", err)
		return
	}
	fmt.Printf("Golang's score is %f now.\n", newScore)

	// 取分数最高的3个
	ret := rdb.ZRevRangeWithScores(ctx, zsetKey, 0, 2).Val()
	for _, z := range ret {
		fmt.Println(z.Member, z.Score)
	}

	// 取95~100分的
	op := &redis.ZRangeBy{
		Min: "95",
		Max: "100",
	}
	ret, err = rdb.ZRangeByScoreWithScores(ctx, zsetKey, op).Result()
	if err != nil {
		fmt.Printf("zrangebyscore failed, err:%v\n", err)
		return
	}
	for _, z := range ret {
		fmt.Println(z.Member, z.Score)
	}
}
```

## 扫描遍历所有Key
使用KEYS prefix:* 获取前缀获取所有key
```go
vals, err := rdb.Keys(ctx, "prefix*").Result()
```

```go
// scanKeysDemo1 按前缀查找所有key示例
func scanKeysDemo1() {
	ctx, cancel := context.WithTimeout(context.Background(), 500*time.Millisecond)
	defer cancel()

	var cursor uint64
	for {
		var keys []string
		var err error
		// 按前缀扫描key
		keys, cursor, err = rdb.Scan(ctx, cursor, "prefix:*", 0).Result()
		if err != nil {
			panic(err)
		}

		for _, key := range keys {
			fmt.Println("key", key)
		}

		if cursor == 0 { // no more keys
			break
		}
	}
}
```

* 简化

```go
// scanKeysDemo2 按前缀扫描key示例
func scanKeysDemo2() {
	ctx, cancel := context.WithTimeout(context.Background(), 500*time.Millisecond)
	defer cancel()
	// 按前缀扫描key
	iter := rdb.Scan(ctx, 0, "prefix:*", 0).Iterator()
	for iter.Next(ctx) {
		fmt.Println("keys", iter.Val())
	}
	if err := iter.Err(); err != nil {
		panic(err)
	}
}
```

* 支持将所有匹配指定的模式key删除的示例
```go
// delKeysByMatch 按match格式扫描所有key并删除
func delKeysByMatch(match string, timeout time.Duration) {
	ctx, cancel := context.WithTimeout(context.Background(), timeout)
	defer cancel()

	iter := rdb.Scan(ctx, 0, match, 0).Iterator()
	for iter.Next(ctx) {
		err := rdb.Del(ctx, iter.Val()).Err()
		if err != nil {
			panic(err)
		}
	}
	if err := iter.Err(); err != nil {
		panic(err)
	}
}
```

* 支持 set 、 hash、zset 数据类型， 遍历

```go
iter := rdb.SScan(ctx, "set-key", 0, "prefix:*", 0).Iterator()
iter := rdb.HScan(ctx, "hash-key", 0, "prefix:*", 0).Iterator()
iter := rdb.ZScan(ctx, "sorted-hash-key", 0, "prefix:*", 0).Iterator(
```

## Pipeline
* Redis Pipeline 允许通过使用单个 client-server-client 往返执行多个命令来提高性能。区别于一个接一个地执行100个命令，你可以将这些命令放入 pipeline 中，然后使用1次读写操作像执行单个命令一样执行它们。这样做的好处是节省了执行命令的网络往返时间（RTT）。

```go
pipe := rdb.Pipeline()

incr := pipe.Incr(ctx, "pipeline_counter")
pipe.Expire(ctx, "pipeline_counter", time.Hour)

cmds, err := pipe.Exec(ctx)
if err != nil {
	panic(err)
}

// 在执行pipe.Exec之后才能获取到结果
fmt.Println(incr.Val())
```
* 使用Pipelined 方法，它会在函数退出时调用 Exec

```go
var incr *redis.IntCmd

cmds, err := rdb.Pipelined(ctx, func(pipe redis.Pipeliner) error {
	incr = pipe.Incr(ctx, "pipelined_counter")
	pipe.Expire(ctx, "pipelined_counter", time.Hour)
	return nil
})
if err != nil {
	panic(err)
}

// 在pipeline执行后获取到结果
fmt.Println(incr.Val())
```

* 遍历 pipeline 命令的返回值依次获取每个命令的结果。下方的示例代码中使用pipiline一次执行了100个 Get 命令，在pipeline 执行后遍历取出100个命令的执行结果。
```go
cmds, err := rdb.Pipelined(ctx, func(pipe redis.Pipeliner) error {
	for i := 0; i < 100; i++ {
		pipe.Get(ctx, fmt.Sprintf("key%d", i))
	}
	return nil
})
if err != nil {
	panic(err)
}

for _, cmd := range cmds {
    fmt.Println(cmd.(*redis.StringCmd).Val())
}
```

## 事务
* 使用 TxPipeline 或 TxPipelined 方法将 pipeline 命令使用 MULTI 和EXEC包裹起来。
```go
// TxPipeline demo
pipe := rdb.TxPipeline()
incr := pipe.Incr(ctx, "tx_pipeline_counter")
pipe.Expire(ctx, "tx_pipeline_counter", time.Hour)
_, err := pipe.Exec(ctx)
fmt.Println(incr.Val(), err)

// TxPipelined demo
var incr2 *redis.IntCmd
_, err = rdb.TxPipelined(ctx, func(pipe redis.Pipeliner) error {
	incr2 = pipe.Incr(ctx, "tx_pipeline_counter")
	pipe.Expire(ctx, "tx_pipeline_counter", time.Hour)
	return nil
})
fmt.Println(incr2.Val(), err)
```

## Watch
* 搭配 WATCH命令来执行事务操作。从使用WATCH命令监视某个 key 开始，直到执行EXEC命令的这段时间里，如果有其他用户抢先对被监视的 key 进行了替换、更新、删除等操作，那么当用户尝试执行EXEC的时候，事务将失败并返回一个错误，用户可以根据这个错误选择重试事务或者放弃事务。

* Watch方法接收一个函数和一个或多个key作为参数。
```go
Watch(fn func(*Tx) error, keys ...string) error
```

*  Watch 方法搭配 TxPipelined 的使用示例
```go
// watchDemo 在key值不变的情况下将其值+1
func watchDemo(ctx context.Context, key string) error {
	return rdb.Watch(ctx, func(tx *redis.Tx) error {
		n, err := tx.Get(ctx, key).Int()
		if err != nil && err != redis.Nil {
			return err
		}
		// 假设操作耗时5秒
		// 5秒内我们通过其他的客户端修改key，当前事务就会失败
		time.Sleep(5 * time.Second)
		_, err = tx.TxPipelined(ctx, func(pipe redis.Pipeliner) error {
			pipe.Set(ctx, key, n+1, time.Hour)
			return nil
		})
		return err
	}, key)
}
```

## 一个 go-redis 官方文档中使用 GET 、SET和WATCH命令实现一个 INCR 命令的完整示例。
```go
const routineCount = 100

increment := func(key string) error {
	txf := func(tx *redis.Tx) error {
		// 获得当前值或零值
		n, err := tx.Get(key).Int()
		if err != nil && err != redis.Nil {
			return err
		}

		// 实际操作（乐观锁定中的本地操作）
		n++

		// 仅在监视的Key保持不变的情况下运行
		_, err = tx.Pipelined(func(pipe redis.Pipeliner) error {
			// pipe 处理错误情况
			pipe.Set(key, n, 0)
			return nil
		})
		return err
	}

	for retries := routineCount; retries > 0; retries-- {
		err := rdb.Watch(txf, key)
		if err != redis.TxFailedErr {
			return err
		}
		// 乐观锁丢失
	}
	return errors.New("increment reached maximum number of retries")
}

var wg sync.WaitGroup
wg.Add(routineCount)
for i := 0; i < routineCount; i++ {
	go func() {
		defer wg.Done()

		if err := increment("counter3"); err != nil {
			fmt.Println("increment error:", err)
		}
	}()
}
wg.Wait()

n, err := rdb.Get("counter3").Int()
fmt.Println("ended with", n, err)
```

* 在这个示例中使用了 redis.TxFailedErr 来检查事务是否失败。

参考链接
https://pkg.go.dev/github.com/go-redis/redis
https://www.liwenzhou.com/posts/Go/redis/#autoid-1-2-0
