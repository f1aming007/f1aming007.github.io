# Golang Cobra

![Minion](https://images.unsplash.com/photo-1483389127117-b6a2102724ae?ixlib=rb-4.0.3&ixid=M3wxMjA3fDB8MHxwaG90by1wYWdlfHx8fGVufDB8fHx8fA%3D%3D&auto=format&fit=crop&w=1548&q=80)# golang-cobra
* **介绍**
    * cobra 是一个命令行程序库
    * 提供了脚手架，用于生成基于cobra的应用程序框架
<!--more-->
## 安装&使用
安装 cobra
```bash
go get github.com/spf13/cobra/cobra
```

> 实现一个git 模拟命令, 通过*os/exec* 库调用外部程序执行真实的git命令
目录结构如下
```bash
tree
.
├── cmd
│   ├── helper.go
│   ├── root.go
│   └── version.go
└── main.go
```
> root.go
```go
package cmd

import (
	"errors"

	"github.com/spf13/cobra"
)

var rootCmd = &cobra.Command{
	Use:   "git",
	Short: "Git is a distributed version control system.",
	Long: `Git is a free and open source distributed version control system
designed to handle everything from small to very large projects 
with speed and efficiency.`,
	Run: func(cmd *cobra.Command, args []string) {
		Error(cmd, args, errors.New("unrecognized command"))
	},
}

func Execute() {
	rootCmd.Execute()
}
```
> version.go
```go
package cmd

import (
	"fmt"
	"os"

	"github.com/spf13/cobra"
)

var versionCmd = &cobra.Command{
	Use:   "version",
	Short: "version subcommand show git version info.",

	Run: func(cmd *cobra.Command, args []string) {
		output, err := ExecuteCommand("git", "version", args...)
		if err != nil {
			Error(cmd, args, err)
		}

		fmt.Fprint(os.Stdout, output)
	},
}

func init() {
	rootCmd.AddCommand(versionCmd)
}
```
> main.go
```go
package main

import (
	"demo/dajun/cobra/cmd"
)

func main() {
	cmd.Execute()
}
```
> helper.go
* 封装了调用外部程序和错误处理函数
```bash
package cmd

import (
	"fmt"
	"os"
	"os/exec"

	"github.com/spf13/cobra"
)

func ExecuteCommand(name string, subname string, args ...string) (string, error) {
	args = append([]string{subname}, args...)

	cmd := exec.Command(name, args...)
	bytes, err := cmd.CombinedOutput()

	return string(bytes), err
}

func Error(cmd *cobra.Command, args []string, err error) {
	fmt.Fprintf(os.Stderr, "execute %s args:%v error:%v\n", cmd.Name(), args, err)
	os.Exit(1)
}
```

* 每个cobra 程序都有一个根命令，可以给他添加任意多个子命令， 在version.go init函数将子命令添加到跟命令中
* 不能直接go run main.go，这已经不是单文件程序了。如果强行要用，请使用go run .

```bash
go build -o main
```

* cobra 自动生成的帮助信息
```bash
./main -h
Git is a free and open source distributed version control system
designed to handle everything from small to very large projects 
with speed and efficiency.

Usage:
  git [flags]
  git [command]

Available Commands:
  completion  Generate the autocompletion script for the specified shell
  help        Help about any command
  version     version subcommand show git version info.

Flags:
  -h, --help   help for git

Use "git [command] --help" for more information about a command
```
* 单个子命令的帮助信息
```bash
./main version -h
version subcommand show git version info.

Usage:
  git version [flags]

Flags:
  -h, --help   help for version
```
* 调用子命令
```bash
./main version
git version 2.39.2 (Apple Git-143)
```

## 特性
cobra 提供非常丰富的功能
* 轻松支持子命令如 **app server, app fetch**等
* 完全兼容POSIX选项（包括短，长选项）；
* 嵌套子命令
* 全局/本地层级选项，可以在多处设置选项， 按照一定的顺序调用
* 使用脚手架轻松胜出程序框架和命令

### 基本概念
* 命令(Command): 需要执行的的操作。
* 参数(Arg) ： 命令的参数， 即要操作的对象。
* 选项(Flag)： 命令选项可以调整命令的行为。

## 命令
在cobra中， 命令和子命令都是用 `Command` 结构表示的，`Command`有非常多的字段，用来制定命令的行为，
* `Use`: 使用信息，即命令怎么被调用，`Example: add [-F file | -D dir]... [-f format] profile`
* `Short\Long`：命令的帮助信息，前者是简短的，后者是详细的
* `Run`： 实际执行的操作函数

定义新的子命令很简单， 创建一个`cobra.Command`变量， 设置一些字段，添加到根命令
> 添加一个clone
```go
package cmd

import (
    "fmt"
    "os"

    "github.com/spf13/cobra"
)

var cloneCmd = &cobra.Command {
    Use: "clone url [destination]",
    Short: "Clone a repository into a new directory",
    Run: func(cmd *cobra.Command, args []string) {
        output, err := ExecuteCommand("git", "clone", args...)
        if err != nil {
            Error(cmd, args, err)
        }
        
        fmt.Fprintf(os.Stdout, output)
    },
}

func init() {
    rootCmd.AddCommand(cloneCmd)
}
```
* `Use`： clone url [destination]
    * `clone`: 子命令
    * `url`： 参数
    * `destination` : 目标路径

## 选项
cobra 的选项分为两种
1. 永久选项：定义的命令和其子命令都可以使用，通过给跟命令添加一个选项定义全局选项 
2. 本地选项：定义它 的命令中使用

cobra 使用`pflag` 解析命令行选项，
存储选项的变量需要提前定义好
```go
var Verbose bool
var Source string
```
设置永久选项
```go
rootCmd.PersistentFlags().BoolVarP(&Verbose, "verbose", "v", false, "verbose output")
```
设置本地选项
```go
localCmd.Flags().StringVarP(&Source, "source", "s", "", "Source directory to read from")
```

> 座一个计算机功能， 支持加、减、乘、除操作。并且可以通过选项设置是否忽略非数字参数，设置除 0 是否报错。 显然，前一个选项应该放在全局选项中，后一个应该放在除法命令中
> 程序结构
```bash
├── cmd
│   ├── add.go
│   ├── divide.go
│   ├── helper.go
│   ├── minus.go
│   ├── multiply.go
│   └── root.go
└── main.go
```
`devide.go`
```go
package cmd

import (
	"fmt"
	"strings"

	"github.com/spf13/cobra"
)

var (
	dividedByZeroHanding int // 除 0 如何处理
)

var divideCmd = &cobra.Command {
	Use: "divide",
	Short: "Divide subcommand divide all passed args.",
	Run: func(cmd *cobra.Command, args []string) {
		values := ConvertArgsToFloat64Slice(args, ErrorHandling(parseHandling))
		result := calc(values, DIVIDE)
		fmt.Printf("%s = %.2f\n", strings.Join(args, "/"), result)
	},
}

func init() {
	divideCmd.Flags().IntVarP(&dividedByZeroHanding, "divide_by_zero", "d", int(PanicOnDividedByZero), "do what when divided by zero")

	rootCmd.AddCommand(divideCmd)
}
```

`root.go`
```go
var (
  parseHandling int
)

var rootCmd = &cobra.Command {
  Use: "math",
  Short: "Math calc the accumulative result.",
  Run: func(cmd *cobra.Command, args []string) {
    Error(cmd, args, errors.New("unrecognized subcommand"))
  },
}

func init() {
  rootCmd.PersistentFlags().IntVarP(&parseHandling, "parse_error", "p", int(ContinueOnParseError), "do what when parse arg error")
}

func Execute() {
  rootCmd.Execute()
}
```

在divide.go中定义了如何处理除 0 错误的选项，在root.go中定义了如何处理解析错误的选项。选项枚举如下：
```go
const (
  ContinueOnParseError  ErrorHandling = 1 // 解析错误尝试继续处理
  ExitOnParseError      ErrorHandling = 2 // 解析错误程序停止
  PanicOnParseError     ErrorHandling = 3 // 解析错误 panic
  ReturnOnDividedByZero ErrorHandling = 4 // 除0返回
  PanicOnDividedByZero  ErrorHandling = 5 // 除0 painc
)
```

测试程序
```bash
$ go build -o math
$ ./math add 1 2 3 4
1+2+3+4 = 10.00

$ ./math minus 1 2 3 4
1-2-3-4 = -8.00

$ ./math multiply 1 2 3 4
1*2*3*4 = 24.00

$ ./math divide 1 2 3 4
1/2/3/4 = 0.04
```

默认情况下， 解析错误被忽略，
```bash
$ ./math add 1 2a 3b 4
1+2a+3b+4 = 5.00

$ ./math divide 1 2a 3b 4
1/2a/3b/4 = 0.25
```

设置解析失败的处理，2 表示退出程序，3 表示 panic
```bash
./math add 1 2a 3b 4 -p 2
invalid number: 2a

$ ./math add 1 2a 3b 4 -p 3
panic: strconv.ParseFloat: parsing "2a": invalid syntax

goroutine 1 [running]:
```

## 脚手架
始化一个新的Go模块：
* 创建一个新目录
* cd进入那个目录
* run go mod init <MODNAME>
```bash
cd $HOME/code 
mkdir myapp
cd myapp
go mod init github.com/spf13/myapp
```

cobra-cli init也可以从子目录运行
* --author标志的作者姓名。例如cobra-cli init --author "Steve Francia spf@spf13.com"

* --license一起使用的许可证，例如cobra-cli init --license apache

### 给项目添加命令
```bash
cobra-cli add serve
cobra-cli add config
cobra-cli add create -p 'configCmd'
```
运行了这三个命令，你就会有一个类似于以下内容的应用程序结构：
```bash
  ▾ app/
    ▾ cmd/
        config.go
        create.go
        serve.go
        root.go
      main.go
```

参考链接
https://github.com/spf13/cobra-cli
https://darjun.github.io/2020/01/17/godailylib/cobra/
