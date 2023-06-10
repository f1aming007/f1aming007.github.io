# 🔥Golang Beradth FirstSearch

![Minion](https://images.unsplash.com/photo-1463171379579-3fdfb86d6285?ixlib=rb-4.0.3&ixid=M3wxMjA3fDB8MHxwaG90by1wYWdlfHx8fGVufDB8fHx8fA%3D%3D&auto=format&fit=crop&w=1740&q=80)
# 🔥Golang算法 - 快速排序（Quicksort）
广度优先算法的核心思想是：从初始节点开始，应用算符生成第一层节点，检查目标节点是否在这些后继节点中，若没有，再用产生式规则将所有第一层的节点逐一扩展，得到第二层节点，并逐一检查第二层节点中是否包含目标节点。若没有，再用算符逐一扩展第二层的所有节点……，如此依次扩展，检查下去，直到发现目标节点为止。即

<!--more-->

## 图（Graph）是什么？
图是用来对不同事物间如何管理进行建模的一种方式
![Minion](http://data.biancheng.net/uploads/allimg/220724/2-220H4153114520.gif)

## 数据结构 Queue
* 现进来的数据先处理（FIFO）
* 无法随机的访问Queue 里面的元素
* 相关操作
    * enqueue: 添加元素
    * dequeue: 移除元素


> 代码示例
```go
package main

import "fmt"

type GraphMap map[string][]string

func main() {

	var graphMap GraphMap = make(GraphMap, 0)
	graphMap["you"] = []string{"alice", "bob", "claire"}
	graphMap["bob"] = []string{"anuj", "peggy"}
	graphMap["alice"] = []string{"peggy"}
	graphMap["claire"] = []string{"tom", "johnny"}
	graphMap["anuj"] = []string{}
	graphMap["peggy"] = []string{}
	graphMap["tom"] = []string{}
	graphMap["johnny"] = []string{}

	searchQueue := graphMap["you"]
	for {
		if len(searchQueue) > 0 {
			var person string
			person, searchQueue = searchQueue[0], searchQueue[1:]
			if personIsTom(person) {
				fmt.Printf("%s is the man\n", person)
				break
			} else {
				searchQueue = append(searchQueue, graphMap[person]...)
			}
		} else {
			fmt.Println("Not Found")
			break
		}
	}
}

func personIsTom(p string) bool {
	return p == "tom"
}
```
> 参考视频
{{< bilibili BBV1rq4y197zx >}}

