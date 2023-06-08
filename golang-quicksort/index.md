# Golang Quicksort

![Minion](https://images.unsplash.com/photo-1685853995266-d095653bc6d1?ixlib=rb-4.0.3&ixid=M3wxMjA3fDB8MHxwaG90by1wYWdlfHx8fGVufDB8fHx8fA%3D%3D&auto=format&fit=crop&w=1742&q=80)
# 🔥Golang算法 - 快速排序（Quicksort）
* 不需要排序的数组（Base case 基线条件）
    * [],空数组
    * [s], 单元素数组
* 容易排序的数组
    * [a,b],两个元素的数组，只需坚持它们之间的大小即可，调整位置
<!--more-->

## 使用Quicksort 排序数组
* 3个元素的数组（例如[23, 19, 35]）
    * 使用D & C策略， 简化为基线条件（Base case）
    1. 从数组中随便找一个元素， 例如35， 这个元素叫做**pivot**(基准元素)
    2. 找到比pivot小的元素，找到比pivot大的元素， 这叫做分区：[23, 19],(35),[]
    3. 如果左右两个子数组已排好序（达到基线条件）， 结果： 左边 + [pivot] + 右边
    4. 如果左右两个子数组没有已排好序（没有达到基线条件）， 那么： quiksort左边 + [pivot] + quicksort右边
    
## 使用Quicksourt  排序数组的步骤
1. 选择一个pivot
2. 将数组分为两个子数组
    1. 左侧数组的元素都比pivot小
    2. 右侧数组的元素都比pivot大
3. 在两个子数组上递归的调用quicksort

> 代码示例01
```go
package main

import "fmt"

func main() {
	arr := []int{12, 87, 62, 66, 30, 12, 328, 121, 67, 98, 3, 256, 81}
	result := quicksort(arr)
	fmt.Println(result)
}

func quicksort(arr []int) []int {
	if len(arr) < 2 {
		return arr
	}
	pivot := arr[0]
	var left, right []int

	for _, ele := range arr[1:] {
		if ele <= pivot {
			left = append(left, ele)
		} else {
			right = append(right, ele)
		}
	}
	return append(quicksort(left), append([]int{pivot}, quicksort(right)...)...)
}
```
> 参考视频
{{< bilibili BV1k34y1S71P >}}
