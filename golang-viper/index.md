# Golang Viper

![Minion](https://images.unsplash.com/photo-1531297484001-80022131f5a1?ixlib=rb-4.0.3&ixid=M3wxMjA3fDB8MHxwaG90by1wYWdlfHx8fGVufDB8fHx8fA%3D%3D&auto=format&fit=crop&w=1640&q=80)
# golang-viper 介绍
viper 是一个配置解决方案，用于丰富的特性
* 支持 JSON/TOML/YAML/HCL/envfile/Java properties 等多种格式的配置文件
* 可以设置监听配置文件的修改，修改时自动加载新的配置；
* 从环境变量、命令行选项和`io.Reader`中读取配置
* 从远程配置系统中读取和监听修改，如 etcd/Consul；
* 代码逻辑中显示设置健值
<!--more-->
## 使用
安装
```bash
go get github.com/spf13/viper
```
使用
```go
package main

import (
  "fmt"
  "log"

  "github.com/spf13/viper"
)

func main() {
  viper.SetConfigName("config")
  viper.SetConfigType("toml")
  viper.AddConfigPath(".")
  viper.SetDefault("redis.port", 6381)
  err := viper.ReadInConfig()
  if err != nil {
    log.Fatal("read config failed: %v", err)
  }

  fmt.Println(viper.Get("app_name"))
  fmt.Println(viper.Get("log_level"))

  fmt.Println("mysql ip: ", viper.Get("mysql.ip"))
  fmt.Println("mysql port: ", viper.Get("mysql.port"))
  fmt.Println("mysql user: ", viper.Get("mysql.user"))
  fmt.Println("mysql password: ", viper.Get("mysql.password"))
  fmt.Println("mysql database: ", viper.Get("mysql.database"))

  fmt.Println("redis ip: ", viper.Get("redis.ip"))
  fmt.Println("redis port: ", viper.Get("redis.port"))
}
```

viper的配置
* `SetConfigName`： 设置文件名
* `SetConfigType`： 配置类型
* `AddConfigPath`： 搜索路径
* `ReadInConfig`： 读取配置
* `viper.Get` ： 获取键值

> ⚠️注意
* 设置文件名时不要带后缀
* 搜索路径可以设置多个，viper会根据设置顺序依次查找
* viper 获取值使用`section.key`的形式，即传入嵌套的键名
* 默认值可以调用 `viper.SetDefault`设置

## 读取键
`GetType` 系列方法可以返回指定类型的值，
* type可以时`Bool/Float64/Int/String/Time/Duration/IntSlice/StringSlice`
* 如果指定的键不存在或者类型不正确，`GetType`方法返回对应类型时零值

判断某个键是否存在， 使用`IsSet`
* `GetStringMap`： 直接以 map 返回某个键下面所有的键值对
* `GetStringMapString`： 返回map[string]string

```go
package main

import (
	"fmt"
	"log"

	"github.com/spf13/viper"
)

func main() {
	viper.SetConfigName("config")
	viper.SetConfigType("toml")
	viper.AddConfigPath(".")
	err := viper.ReadInConfig()
	if err != nil {
		log.Fatalf("read config failed: %v", err)
	}

	fmt.Println("protocols: ", viper.GetStringSlice("server.protocols"))
	fmt.Println("ports: ", viper.GetIntSlice("server.ports"))
	fmt.Println("timeout: ", viper.GetDuration("server.timeout"))

	fmt.Println("mysql ip: ", viper.GetString("mysql.ip"))
	fmt.Println("mysql port: ", viper.GetInt("mysql.port"))

	if viper.IsSet("redis.port") {
		fmt.Println("redis.port is set")
	} else {
		fmt.Println("redis.port is not set")
	}

	fmt.Println("mysql settings: ", viper.GetStringMap("mysql"))
	fmt.Println("redis settings: ", viper.GetStringMap("redis"))
	fmt.Println("all settings: ", viper.AllSettings())
}
```

## 设置键值
viper 支持在多个地方设置，使用下面的顺序依次读取
* 调用 `Set` 显示设置的
* 命令行选项；
* 环境变量
* 默认值

> viper.Set
如果某个键通过 `viper.set`  设置了值， 这个值优先级最高
```go
viper.Set("redis.port", 5381)
```

## 命令行选项
viper使用pflag库来解析选项，首先在 `init` 方法中定义选项， 并且在`viper.BindPFlags` 绑定选项来配置中
```go
func init() {
  pflag.Int("redis.port", 8381, "Redis port to connect")

  // 绑定命令行
  viper.BindPFlags(pflag.CommandLine)
}
```

## 环境变量
在init方法中调用AutomaticEnv方法绑定全部环境变量
```go
func init() {
  // 绑定环境变量
  viper.AutomaticEnv()
}
```
单独绑定环境变量
```go
func init() {
  // 绑定环境变量
  viper.BindEnv("redis.port")
  viper.BindEnv("go.path", "GOPATH")
}

func main() {
  // 省略部分代码
  fmt.Println("go path: ", viper.Get("go.path"))
}
```
* 调用 `BindEnv` 方法
    * 只传一个参数， 即表示键名，又表示环境变量名
    * 传入两个参数， 第一个参数表示键名，第二个表示环境变量名


## 读取配置

### 从`io.Reader` 中读取
viper 支持从 `io.Reader` 中读取配置， 配置灵活
* 文件
* 程序中生成的字符串
* 从网络中链接中读取的字节流
```go
package main

import (
  "bytes"
  "fmt"
  "log"

  "github.com/spf13/viper"
)

func main() {
  viper.SetConfigType("toml")
  tomlConfig := []byte(`
app_name = "awesome web"

# possible values: DEBUG, INFO, WARNING, ERROR, FATAL
log_level = "DEBUG"

[mysql]
ip = "127.0.0.1"
port = 3306
user = "dj"
password = 123456
database = "awesome"

[redis]
ip = "127.0.0.1"
port = 7381
`)
  err := viper.ReadConfig(bytes.NewBuffer(tomlConfig))
  if err != nil {
    log.Fatal("read config failed: %v", err)
  }

  fmt.Println("redis port: ", viper.GetInt("redis.port"))
}
```

### `Unmarshal`
viper 支持将配置Unmarshal到一个结构体中，为结构体中的对应字段赋值
```go
package main

import (
  "fmt"
  "log"

  "github.com/spf13/viper"
)

type Config struct {
  AppName  string
  LogLevel string

  MySQL    MySQLConfig
  Redis    RedisConfig
}

type MySQLConfig struct {
  IP       string
  Port     int
  User     string
  Password string
  Database string
}

type RedisConfig struct {
  IP   string
  Port int
}

func main() {
  viper.SetConfigName("config")
  viper.SetConfigType("toml")
  viper.AddConfigPath(".")
  err := viper.ReadInConfig()
  if err != nil {
    log.Fatal("read config failed: %v", err)
  }

  var c Config
  viper.Unmarshal(&c)

  fmt.Println(c.MySQL)
}
```

## 保存配置
将程序中生成的配置，或者所做的修改保存下来
* `WriteConfig`： 将当前的viper配置写到预定义路径， 如果没有预定义路径，返回错误，将会覆盖当前配置
* `SafeWriteConfig`： 与上面功能一样，如果配置文件存在，则不覆盖
* `WriteConfigAs`： 保存配置到指定路径，如果文件存在，则覆盖
* `SafeWriteConfig`： 与上面功能一样，但是入股配置文件存在，则不覆盖

```go
package main

import (
  "log"

  "github.com/spf13/viper"
)

func main() {
  viper.SetConfigName("config")
  viper.SetConfigType("toml")
  viper.AddConfigPath(".")

  viper.Set("app_name", "awesome web")
  viper.Set("log_level", "DEBUG")
  viper.Set("mysql.ip", "127.0.0.1")
  viper.Set("mysql.port", 3306)
  viper.Set("mysql.user", "root")
  viper.Set("mysql.password", "123456")
  viper.Set("mysql.database", "awesome")

  viper.Set("redis.ip", "127.0.0.1")
  viper.Set("redis.port", 6381)

  err := viper.SafeWriteConfig()
  if err != nil {
    log.Fatal("write config failed: ", err)
  }
}
```

## 监听文件修改
viper 可以监听文件修改， 热加载配置，不需要重启服务器， 让配置生效
```go
package main

import (
  "fmt"
  "log"
  "time"

  "github.com/spf13/viper"
)

func main() {
  viper.SetConfigName("config")
  viper.SetConfigType("toml")
  viper.AddConfigPath(".")
  err := viper.ReadInConfig()
  if err != nil {
    log.Fatal("read config failed: %v", err)
  }

  viper.WatchConfig()

  fmt.Println("redis port before sleep: ", viper.Get("redis.port"))
  time.Sleep(time.Second * 10)
  fmt.Println("redis port after sleep: ", viper.Get("redis.port"))
}
```

`viper.WatchConfig`：  viper 会自动监听配置修改，如果有修改，重新加载的配置

为配置修改增加一个回调
```go
viper.OnConfigChange(func(e fsnotify.Event) {
  fmt.Printf("Config file:%s Op:%s\n", e.Name, e.Op)
})
```

> 参考链接
https://darjun.github.io/2020/01/18/godailylib/viper/
