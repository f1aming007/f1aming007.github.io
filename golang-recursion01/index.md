# 🔥Golang Recursion


{{< figure src="/images/Recursion02.png" title="go-work (figure)" >}}
# 🔥Golang算法 - 递归

* 递归： 在运行的过程中调用自己
<!--more-->

## 递归函数 关键点
* 在同一函数调用的函数
* 在递归函数中指定一个退出条件总是好的
* 它可以检查不必要的函数调用和代码行
* 递归的缺点 代码逻辑性强， 难以调试和代码检查

{{< figure src="/images/Recursion01.png" title="go-work (figure)" >}}

> 示例
```go
package main

import "fmt"

func main() {
	doll := Item{
		ID:   1,
		Type: "doll",
		Child: &Item{
			ID:   2,
			Type: "doll",
			Child: &Item{
				ID:   3,
				Type: "doll",
				Child: &Item{
					ID:    4,
					Type:  "diamond",
					Child: nil,
				},
			},
		},
	}

	diamond := findDiamond(doll)
	fmt.Printf("Item %d is diamond\n", diamond.ID)

}

func findDiamond(item Item) Item {
	if item.IsDoll() {
		return findDiamond(*item.Child)
	} else {
		return item
	}
}

type Item struct {
	ID    int
	Type  string
	Child *Item
}

type ItemClassifier interface {
	IsDoll() bool
}

func (it *Item) IsDoll() bool {
	if it.Type == "doll" {
		return true
	}
	return false
}
```

> 输出结果
```bash
go run main.go
Item 4 is diamond
```

> 示例二 （递归函数实现斐波那契数）

```go
package main

import "fmt"

func fibonacci(n int) int {
  if n < 2 {
   return n
  }
  return fibonacci(n-2) + fibonacci(n-1)
}

func main() {
    var i int
    for i = 0; i < 10; i++ {
       fmt.Printf("%d\t", fibonacci(i))
    }
}
```
