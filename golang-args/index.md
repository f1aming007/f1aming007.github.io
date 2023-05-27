# Golang 命令行参数os.Args

![Minion](https://images.unsplash.com/photo-1629654297299-c8506221ca97?ixlib=rb-4.0.3&ixid=M3wxMjA3fDB8MHxwaG90by1wYWdlfHx8fGVufDB8fHx8fA%3D%3D&auto=format&fit=crop&w=1548&q=80)
# Golang 命令行参数os.Args
* 如何制作命令行应用
* 如何使用os.Args 获取命令行参数
<!--more-->
## 使用的知识点

* os包提供了用于处理操作系统的相关内容的函数/值
    * 独立于平台的方式

* os.Args 变量
    * 获取命令行的参数
    * 它是string slice
    * 第一个值是命令本身

* strings.Join 函数
    * 将字符串切片中存在的所有元素连接为单个字符串

> 示例01
```go
package main

import (
	"fmt"
	"os"
)

func main() {
	var s, sep string

	for i := 0; i < len(os.Args); i++ {
		s += sep + os.Args[i]
		sep = " "
	}

	fmt.Println(s)
}

```

> 示例02
```go
package main

import (
	"fmt"
	"os"
)

func main() {
	s, sep := "", ""

	for _, args := range os.Args[1:] {
		s += sep + args
		sep = " "
	}

	fmt.Println(s)
}
```
> 示例03
```go
package main

import (
	"fmt"
	"os"
	"strings"
)

func main() {
	fmt.Println(strings.Join(os.Args[1:], " "))
}
```

## 用户输入bufio
```go
package main

import (
	"bufio"
	"fmt"
	"os"
)

func main() {
	fmt.Println("Waht's your name")
	reader := bufio.NewReader(os.Stdin)
	text, _ := reader.ReadString('\n')
	fmt.Printf("your name: %s", text)
}
```
