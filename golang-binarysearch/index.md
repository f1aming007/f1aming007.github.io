# 🚀Golang BinarySearch

![Minion](https://plus.unsplash.com/premium_photo-1668461477148-3624cd4f7b71?ixlib=rb-4.0.3&ixid=M3wxMjA3fDB8MHxwaG90by1wYWdlfHx8fGVufDB8fHx8fA%3D%3D&auto=format&fit=crop&w=1642&q=80)
# 🔥Golang算法- 二分算法（Binary Search）
* 输入：**排好序** 的集合
* 如果要查询的元素在集合中： 返回位置（索引）
* 否则：返回空
<!--more-->

## Binary Search 介绍
* 针对拥有n个元素的一排序的集合
    * 二分查找： ``log2n``
    * 简单查找： n
    
> ⚠️注意
* 二分查找只适用于**已排序**的集合

> 示例

```go
package main

import (
	"fmt"
	"math/rand"
	"time"
)

func main() {
	list := make([]int, 1_000_000)
	for i := 0; i < 1_000_000; i++ {
		list = append(list, i+1)
	}
	rand.Seed(time.Now().UnixNano())
	for i := 0; i < 20; i++ {
		v := rand.Intn(1_000_000-1) + 1
		fmt.Printf("针对 %d 进行二分查找： \n", v)
		idx := binarySearch(list, v)
		fmt.Printf("%d 的索引位置是：[%d]\n", v, idx)
		fmt.Println("-------------------------------")
	}
}

func binarySearch(list []int, target int) int {
	low := 0
	high := len(list)

	step := 0
	for {
		step = step + 1
		if low <= high {
			mid := (low + high) / 2
			guess := list[mid]
			if guess == target {
				fmt.Printf("共查找了 %d 次\n", step)
				return mid
			}
			if guess > target {
				high = mid + 1
			} else {
				low = mid - 1
			}
		}
	}
}

```

> 输出结果
```bash
go run main.go
针对 488291 进行二分查找： 
共查找了 21 次
488291 的索引位置是：[1488290]
-------------------------------
针对 253611 进行二分查找： 
共查找了 19 次
253611 的索引位置是：[1253610]
-------------------------------
针对 92000 进行二分查找： 
共查找了 21 次
92000 的索引位置是：[1091999]
-------------------------------
针对 399702 进行二分查找： 
共查找了 17 次
399702 的索引位置是：[1399701]
-------------------------------
针对 356459 进行二分查找： 
共查找了 17 次
356459 的索引位置是：[1356458]
-------------------------------
针对 533248 进行二分查找： 
共查找了 17 次
533248 的索引位置是：[1533247]
-------------------------------
针对 802470 进行二分查找： 
共查找了 21 次
802470 的索引位置是：[1802469]
-------------------------------
针对 358966 进行二分查找： 
共查找了 21 次
358966 的索引位置是：[1358965]
-------------------------------
针对 325215 进行二分查找： 
共查找了 21 次
325215 的索引位置是：[1325214]
-------------------------------
针对 443265 进行二分查找： 
共查找了 21 次
443265 的索引位置是：[1443264]
-------------------------------
针对 829382 进行二分查找： 
共查找了 18 次
829382 的索引位置是：[1829381]
-------------------------------
针对 499513 进行二分查找： 
共查找了 21 次
499513 的索引位置是：[1499512]
-------------------------------
针对 589792 进行二分查找： 
共查找了 19 次
589792 的索引位置是：[1589791]
-------------------------------
针对 253817 进行二分查找： 
共查找了 21 次
253817 的索引位置是：[1253816]
-------------------------------
针对 624237 进行二分查找： 
共查找了 20 次
624237 的索引位置是：[1624236]
-------------------------------
针对 850665 进行二分查找： 
共查找了 21 次
850665 的索引位置是：[1850664]
-------------------------------
针对 929880 进行二分查找： 
共查找了 21 次
929880 的索引位置是：[1929879]
-------------------------------
针对 422010 进行二分查找： 
共查找了 20 次
422010 的索引位置是：[1422009]
-------------------------------
针对 39939 进行二分查找： 
共查找了 21 次
39939 的索引位置是：[1039938]
-------------------------------
针对 894763 进行二分查找： 
共查找了 19 次
894763 的索引位置是：[1894762]
-------------------------------
```

{{< bilibili BV1Hg411G7dc >}}
