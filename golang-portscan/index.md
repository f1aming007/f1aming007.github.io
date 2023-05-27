# Golang PortScan

![Minion](https://images.unsplash.com/photo-1498084393753-b411b2d26b34?ixlib=rb-4.0.3&ixid=M3wxMjA3fDB8MHxwaG90by1wYWdlfHx8fGVufDB8fHx8fA%3D%3D&auto=format&fit=crop&w=1932&q=80)
# Golang TCP 端口扫描
<!--more-->
> 非并发版
```go
package main

import (
	"fmt"
	"net"
)

func main() {
	for i := 21; i < 120; i++ {
		address := fmt.Sprintf("192.168.0.2:%d", i)
		conn, err := net.Dial("tcp", address)
		if err != nil {
			fmt.Printf("%d 关闭了\n", address)
			continue
		}
		conn.Close()
		fmt.Printf("%d dakai了\n", address)
	}
}
```


> 并发版
```go
package main

import (
	"fmt"
	"net"
	"sync"
	"time"
)

func main() {
	start := time.Now()
	var wg sync.WaitGroup
	for i := 21; i < 120; i++ {
		wg.Add(1)
		go func(j int) {
			address := fmt.Sprintf("192.168.0.2:%d", j)
			conn, err := net.Dial("tcp", address)
			if err != nil {
				fmt.Printf("%d 关闭了\n", address)
				return
			}
			conn.Close()
			fmt.Printf("%d dakai了\n", address)
		}(i)
		wg.Wait()
		elapsed := time.Since(start) / 1e9
		fmt.Printf("\n\n%d\n", elapsed)
	}
}
```


> goroutine 池并发版 TCP 端口扫描器
{{< figure src="/images/go-work.png" title="go-work (figure)" >}}

```go
package main

import (
	"fmt"
	"net"
	"sort"
)

func worker(ports chan int, results chan int) {
	for p := range ports {
		address := fmt.Sprintf("192.168.1.1:%d", p)
		conn, err := net.Dial("tcp", address)
		if err != nil {
			results <- 0
			fmt.Printf("%d 关闭了 \n", p)
		}
		conn.Close()
		fmt.Printf("%d 打开了", p)
		results <- p
	}
}

func main() {
	ports := make(chan int, 100)
	results := make(chan int)
	var openports []int
	var closeports []int
	for i := 0; i < cap(ports); i++ {
		go worker(ports, results)
	}

	go func() {
		for i := 1; i < 1024; i++ {
			ports <- i
		}	
	}()
	
	for i := 1; i < 1024; i++ {
		port := <- results
		if port != 0 {
			openports = append(openports, port)
		} else {
			closeports = append(closeports, port)
		}
	}
	
	close(ports)
	close(results)
	
	sort.Ints(openports)
	sort.Ints(closeports)
	
	for _, port := range openports {
		fmt.Printf("%d open\n", port)
	}

	for _, port := range closeports {
		fmt.Printf("%d close\n", port)
	}
}
```

参考视频

{{< bilibili BV13K4y1a7dt >}}
