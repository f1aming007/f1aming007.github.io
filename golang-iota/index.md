# Golang Iota


# Iota 介绍
* 在常量声明中，预先声明的标识符`iota`代表连续的无类型的整数常量， 他的值是该常量声明中对应`ConstSpec`的索引，从零开始计数。
* iota： 可以在常量声明中自动创建一系列连续的整数值，值从零开始， 不需要手动指定每个常量的值

<!--more-->

## iota的应用场景

### 自动生成递增的常量值
* iota 可以方便生成递增的常量值， 在常量声明中的第一个使用`iota`的常量初始化未0， 而后的常量都会自动递增，这是得在定义一组递增常量时无需手动指定每个常量的值， 提高了代码的 `可读性`和`维护性` 

```go
const (
	Apple  = iota // 0
	Banana        // 1
	Cherry        // 2
)
```

### 构建枚举类型常量
* 使用`iota` 可以轻松定义一系列相关的枚举值， 而无需为每个值手动指定具体的数字，这样的枚举类型定义更加简洁，易于扩展和修改
```go
type WeekDay int

const (
	Sunday    WeekDay = iota // 0
	Tuesday                  // 1
	Wednesday                // 2
	Thursday                 // 3
	Friday                   // 4
	Saturday                 // 5
	Monday                   // 6
)
```

### 表达式计算
* 通过在常量声明中使用`iota`， 可以创建复杂的表达式， 并在每个常量表达式中， 根据需要调整`iota`的值， 可以轻松生成一组具有特定规律的常量
```go
const (
	_  = iota
	KB = 1 << (10 * iota) // 1 << (10 * 1) = 1024B = 1KB
	MB = 1 << (10 * iota) // 1 << (10 * 2) = 1048576B = 1MB
	GB = 1 << (10 * iota) // 1 << (10 * 3) = 1073741824B = 1GB
	TB = 1 << (10 * iota) // 1 << (10 * 4) = 1099511627776B = 1TB
)
```

### 位运算
通过左移运算符 （<<）与iota配合使用，方便地生成一组按位运算的常量
```go
const (
	FlagNone  = 0          // 0
	FlagRead  = 1 << iota  // 1
	FlagWrite              // 2
	FlagExec               // 4
)
```

## Iota 的使用技巧和注意事项

### 跳值使用
使用`_`下划线来忽略某些值，
```go
const (
	Apple = iota // 0
	_
	Banana // 2
)
```

### 不同常量块，iota是独立的
`iota` 的作用范围是整个常量块， 不同常量块的`iota`是独立的， 每个常量块中第一个`iota`的值都是0
```go
const (
	A = iota    // 0
	B        // 1
)

const (
	C = iota    // 0
	D        // 1
)
```
