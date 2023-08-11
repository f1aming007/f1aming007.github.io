# Golang Jsonparser


# Jsonparser 开源的JSON包
号称比JSON包性能高10倍，内存分配优化到0，提高JSON操作的性能
`jsonparser` 的性能在很大程度上取决于使用情况，当不需要处理完成记录而只需要处理某几个字段时（尤其是访问第三方接口时，大多数情况下只需要几个字段）性能表现的非常好

## 方法对应关系
`jsonparser`标准库JSON工作方式不同，不会 `编码/解码` 整个数据结构， 而是按需操作


|      | 标准库    | jsonparser |
|------|-----------|------------|
| 编码 | Marshal   | Set        |
| 解码 | Unmarshal | Get        |

* `jsonparser` 在Get 的基础上封装了很多针对单个字段的使用方法，例如

```golang
package main

import (
	"fmt"
	"log"

	"github.com/buger/jsonparser"
)

var (
	// JSON 字符串
	dataJson = []byte(`
{
  "person": {
    "name": {
      "first": "Leonid",
      "last": "Bugaev",
      "fullName": "Leonid Bugaev"
    },
    "github": {
      "handle": "buger",
      "followers": 109
    },
    "avatars": [
      { "url": "https://avatars1.githubusercontent.com/u/14009?v=3&s=460", "type": "thumbnail" }
    ]
  },
  "company": {
    "name": "Acme"
  }
}
`)
)

func main() {
	// 解析对象
	github := struct {
		Handle    string `json:"handle"`
		Followers int    `json:"followers"`
	}{}

	err := jsonparser.ObjectEach(dataJson, func(key []byte, value []byte, dataType jsonparser.ValueType, offset int) error {
		switch string(key) {
		case "handle":
			github.Handle = string(value)
		case "followers":
			followers, _ := jsonparser.ParseInt(value)
			github.Followers = int(followers)
		}
		return nil
	}, "person", "github")
	if err != nil {
		log.Fatal(err)
	}

	fmt.Printf("github = %+v\n\n", github)

	// 编码结构体
	githubJson, err := jsonparser.Set([]byte(`{}`), []byte(fmt.Sprintf(`{"handle: %s", "followers": "%d"}`, github.Handle, github.Followers)), "github")
	if err != nil {
		log.Fatal(err)
	}
	fmt.Printf("github json = %s\n\n", githubJson)

	// 解析多个 key
	paths := [][]string{
		{"person", "name", "fullName"},
		{"person", "avatars", "[0]", "url"},
		{"company", "name"},
	}

	jsonparser.EachKey(dataJson, func(i int, bytes []byte, valueType jsonparser.ValueType, err error) {
		switch i {
		case 0:
			fmt.Printf("fullName = %s\n", bytes)
		case 1:
			fmt.Printf("avatars[0].url = %s\n", bytes)
		case 2:
			fmt.Printf("company.name = %s\n\n", bytes)
		}
	}, paths...)

	// 解析整数
	n, err := jsonparser.GetInt(dataJson, "person", "github", "followers")
	if err != nil {
		log.Fatal(err)
	}
	fmt.Printf("n = %d\n\n", n)

	// 解析字符串
	name, err := jsonparser.GetString(dataJson, "company", "name")
	if err != nil {
		log.Fatal(err)
	}
	fmt.Printf("name = %s\n", name)
}
```

## 代码实现
`jsonparser` 的核心代码全部放在了一个文件中 `parser.go`, 我们主要关注两种操作的实现: 
`解码GET` 和 `编码Set`

### 数据类型
`jsonparser` 将合法的JSON数据类型简单进行常量映射
```go
// JSON 数据类型常量
type ValueType int

const (
 NotExist = ValueType(iota)
 String
 Number
 Object
 Array
 Boolean
 Null
 Unknown
)
```
为了规避 GC, 用于 JSON 字符串转义分配的 `[]byte` 切片容量上限，超过这个上限值后，转义结果会被分配到 堆上 引发 GC， 也就是 []byte 切片的长度超过 64 之后，会被分配到 堆上 
```go
// 规避 GC 的切片容量上限常量
const unescapeStackBufSize = 64
```

## 辅助方法
* stringEnd 尝试寻找当前字符串的结尾，也就是 ", 支持转义的情况，例如 \"
```go
func stringEnd(data []byte) (int, bool) {}
```

* blockEnd 尝试寻找当前数组或对象的结尾，数组的开始和结尾表示为 [ 和 ], 对象的开始和结尾表示为 { 和 }。
```go
func blockEnd(data []byte, openSym byte, closeSym byte) int {}
```

## searchKeys 状态机
searchKeys 方法是整个 解析操作 的核心方法，内部实现类似于 有限状态机 机制，通过将 JSON 字符串作为输入参数，并根据定义的状态转换规则逐个解析字符， 结果返回解析到的索引，或者 -1 (解析失败)。

### 状态机组成部分
* 输入参数： 存储解析的JSON字符串
* 解析器： 解析 JSON 字符串并返回相应的数据结构 (`searchKeys` 方法返回的是 `[]byte`)
* 状态转移表： 定义状态转换规则，包括当前状态、下一个字符以及下一个状态 (`searchKeys` 主要是通过上面提到的辅助方法来完成的)
* 状态栈 : 记录状态转换过程中的状态 (`searchKeys` 用了几个变量来记录状态，例如 `keyLevel`, `level`, `ln`, `lk` 等)

![](media/16917184230445.jpg)

