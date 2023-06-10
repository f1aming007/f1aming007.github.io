# 🔥Golang Divide_Conquer


![Minion](https://images.unsplash.com/photo-1685538382974-a0b07102ef16?ixlib=rb-4.0.3&ixid=M3wxMjA3fDB8MHxwaG90by1wYWdlfHx8fGVufDB8fHx8fA%3D%3D&auto=format&fit=crop&w=1742&q=80)

# 🔥Golang算法 - D&C 算法
* Devide & Conquer
> D & C 的步骤
1. 找到一个简单的基线条件（Base Case）
2. 把问题分开处理。直到它变为极限条件
<!--more-->
## 例子
* 需求： 将数组[1,3,5,7,9]求和
* 思路1： 使用循环（例如for循环）
* 思路2:  D & C策略
    * 基线条件： 空数组[], 其和为0，
    * 递归：[1,3,5,7,9]
        * 1 + SUM([3,5,7,9])
            * 3 + SUM([5,7,9])
                * 5 + SUM([7,9])
                    * 7 + SUM([9])
                        * 9 + SUM([])

## 代码示例
```go
package main

import "fmt"

func main() {
	totals := sum([]int{1, 3, 5, 7, 9})
	fmt.Println(totals)
}

func sum(arr []int) int {
	if len(arr) == 0 {
		return 0
	}
	return arr[0] + sum(arr[1:])
}
```

> 参考视频
{{< bilibili BV1Eq4y1N7jR >}}
