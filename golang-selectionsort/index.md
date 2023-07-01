# 🔥Golang Selection Sort


    {{< figure src="/images/golang-selection.png" title="go-work (figure)" >}}
# 🔥Golang算法- 选择排序（Selection Sort)
* 在未排序序列中找到最小（大）元素，存放到排序序列的起始位置，
* 然后，再从剩余未排序元素中继续寻找最小（大）元素，
* 然后放到已排序序列的末尾。
* 以此类推，直到所有元素均排序完毕。
<!--more-->

> 示例
```go
package main

import "fmt"

func main() {
	arr := []int{5, 7, 1, 8, 3, 2, 6, 4, 9}
	arr = selectionSort(arr)

	fmt.Println(arr)
}

func findSmallest(arr []int) int {
	smallest := arr[0]
	smallestIndex := 0
	for i := 0; i < len(arr); i++ {
		if arr[i] < smallest {
			smallest = arr[i]
			smallestIndex = i
		}
	}
	return smallestIndex
}

func selectionSort(arr []int) []int {
	result := []int{}
	count := len(arr)
	for i := 0; i < count; i++ {
		smallestIndex := findSmallest(arr)
		result = append(result, arr[smallestIndex])
		arr = append(arr[:smallestIndex], arr[smallestIndex+1:]...)
	}
	return result
}
```

> 输出结果
```bash
go run main.go
[1 2 3 4 5 6 7 8 9]
```

{{< bilibili BV1xy4y1G7tz >}}
