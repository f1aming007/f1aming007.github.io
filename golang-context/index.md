# Golang Context 底层原理

{{< figure src="/images/context01.png" title="go-work (figure)" >}}
# Golang context 实现原理
* context的主要功能
    * 异步场景中用于实现并发协调
    * 对goroutine 的生命周期控制
    * 有一定的数据存储能力

<!--more-->
>  context 的数据结构
```go
type Context interface {
    Deadline() (deadline time.Time, ok bool)
    Done() <-chan struct{}
    Err() error
    Value(key any) any
}
```

**Context 为interface， 定义了四个核心api**
* Deadline： 返回context的过期时间
* Done： 返回context中的channel
* Err： 返回错误
* value： 返回context中的对应的key值

### 标准 error
```go
var Canceled = errors.New("context canceled")

var DeadlineExceeded error = deadlineExceededError{}

type deadlineExceededError struct{}

func (deadlineExceededError) Error() string   { return "context deadline exceeded" }
func (deadlineExceededError) Timeout() bool   { return true }
func (deadlineExceededError) Temporary() bool { return true
```
* canceled： context 被canel时会报此错误
* DeadlineExceeded: context 超时时会报此错误


### emptyCtx 类的实现
```go
type emptyCtx int

func (*emptyCtx) Deadline() (deadline time.Time, ok bool) {
    return
}

func (*emptyCtx) Done() <-chan struct{} {
    return nil
}

func (*emptyCtx) Err() error {
    return nil
}

func (*emptyCtx) Value(key any) any {
    return 
}
```
* emptyCtx时一个空的context， 本质上类型为一个整型
* Deadline 方法会返回一个公元元年时间以及 false 的 flag，标识当前 context 不存在过期时间；
* Done： 方法返回一个nil 值，用户无论在nil中写入或者读取数据，都会陷入阻塞
* Err： 方法返回的错误永远为nil
* Value： 方法返回的value同样永远为nil

### context.Background() & context.TODO()
```go
var (
    background = new(emptyCtx)
    todo       = new(emptyCtx)
)

func Background() Context {
    return background
}

func TODO() Context {
    return todo
}
```
* 常用的 context.Background() 和 context.TODO() 方法返回的均是 emptyCtx 类型的一个实例.


## cancelCtx 数据结构

{{< figure src="/images/context02.png" title="go-work (figure)" >}}

```go
type cancelCtx struct {
    Context
   
    mu       sync.Mutex            // protects following fields
    done     atomic.Value          // of chan struct{}, created lazily, closed by first cancel call
    children map[canceler]struct{} // set to nil by the first cancel call
    err      error                 // set to non-nil by the first cancel call
}

type canceler interface {
    cancel(removeFromParent bool, err error)
    Done() <-chan struct{}
}
```
* embed 了context作为其父context， cancelCtx必然为某个context的子context
* 内置了一把锁， 用以协调并发场景下的资源获取
* done： 实际类型为chan struct{},  即用以反映 cancelCtx生命周期的通道
* children： 一个set 指cancelCtx 的所有子context
* err： 记录了当前cancelCtx的错误， 必然为某一个context的子context

### Deadline 方法
cancelCtx 未实现该方法， 仅时embed了一个带有 Deadline 方法的Context interface 因此直接 调用会报错

### Done 方法
{{< figure src="/images/context03.png" title="go-work (figure)" >}}

```go
func (c *cancelCtx) Done() <-chan struct{} {
    d := c.done.Load()
    if d != nil {
        return d.(chan struct{})
    }
    c.mu.Lock()
    defer c.mu.Unlock()
    d = c.done.Load()
    if d == nil {
        d = make(chan struct{})
        c.done.Store(d)
    }
    return d.(chan struct{})
}
```
* 基于atomic 包， 读取 cancelCtx中的chan； 倘若已存在，则直接返回
* 加锁后， 在此检查chan是否存在， 若存在则返回， 
* 初始化 chan 存储到 aotmic.Value 当中，并返回.（懒加载机制）

### Err 方法
```go
func (c *cancelCtx) Err() error {
    c.mu.Lock()
    err := c.err
    c.mu.Unlock()
    return err
}
```
* 加锁；
* 读取 cancelCtx.err；
* 解锁
* 返回结果

### Value方法
```go
func (c *cancelCtx) Value(key any) any {
    if key == &cancelCtxKey {
        return c
    }
    return value(c.Context, key)
}
```
* 倘若 key 特定值 &cancelCtxKey，则返回 cancelCtx 自身的指针；
* 否则遵循 valueCtx 的思路取值返回

## context.WithCancel()
```go
func WithCancel(parent Context) (ctx Context, cancel CancelFunc) {
    if parent == nil {
        panic("cannot create context from nil parent")
    }
    c := newCancelCtx(parent)
    propagateCancel(parent, &c)
    return &c, func() { c.cancel(true, Canceled) }
}
```
* 校验父 context 非空；
* 注入父 context 构造好一个新的 cancelCtx；
* 在 propagateCancel 方法内启动一个守护协程，以保证父 context 终止时，该 cancelCtx 也会被终止；
* 将 cancelCtx 返回，连带返回一个用以终止该 cancelCtx 的闭包函数.

### newCancelCtx
```go
 func newCancelCtx(parent Context) cancelCtx {
    return cancelCtx{Context: parent}
}
```

### propagateCancel
{{< figure src="/images/context04.png" title="go-work (figure)" >}}

```go
func propagateCancel(parent Context, child canceler) {
	done := parent.Done()
	if done == nil {
		return // parent is never canceled
	}

	select {
	case <-done:
		// parent is already canceled
		child.cancel(false, parent.Err())
		return
	default:
	}

	if p, ok := parentCancelCtx(parent); ok {
		p.mu.Lock()
		if p.err != nil {
			// parent has already been canceled
			child.cancel(false, p.err)
		} else {
			if p.children == nil {
				p.children = make(map[canceler]struct{})
			}
			p.children[child] = struct{}{}
		}
		p.mu.Unlock()
	} else {
		atomic.AddInt32(&goroutines, +1)
		go func() {
			select {
			case <-parent.Done():
				child.cancel(false, parent.Err())
			case <-child.Done():
			}
		}()
	}
}
```

propagateCancel 用以传递父子 context 之间的 cancel 事件：

• 倘若 parent 是不会被 cancel 的类型（如 emptyCtx），则直接返回；
• 倘若 parent 已经被 cancel，则直接终止子 context，并以 parent 的 err 作为子 context 的 err；
• 假如 parent 是 cancelCtx 的类型，则加锁，并将子 context 添加到 parent 的 children map 当中；
• 假如 parent 不是 cancelCtx 类型，但又存在 cancel 的能力（比如用户自定义实现的 context），则启动一个协程，通过多路复用的方式监控 parent 状态，倘若其终止，则同时终止子 context，并透传 parent 的 err.

## cancelCtx.cancel
{{< figure src="/images/context05.png" title="go-work (figure)" >}}

```go
func (c *cancelCtx) cancel(removeFromParent bool, err error) {
    if err == nil {
        panic("context: internal error: missing cancel error")
    }
    c.mu.Lock()
    if c.err != nil {
        c.mu.Unlock()
        return // already canceled
    }
    c.err = err
    d, _ := c.done.Load().(chan struct{})
    if d == nil {
        c.done.Store(closedchan)
    } else {
        close(d)
    }
    for child := range c.children {
        // NOTE: acquiring the child's lock while holding parent's lock.
        child.cancel(false, err)
    }
    c.children = nil
    c.mu.Unlock()

    if removeFromParent {
        removeChild(c.Context, c)
    }
}
```
* cancelCtx.cancel 方法有两个入参，第一个 removeFromParent 是一个 bool 值，表示当前 context 是否需要从父 context 的 children set 中删除；第二个 err 则是 cancel 后需要展示的错误；
* 进入方法主体，首先校验传入的 err 是否为空，若为空则 panic；
* 加锁；
* 校验 cancelCtx 自带的 err 是否已经非空，若非空说明已被 cancel，则解锁返回；
* 将传入的 err 赋给 cancelCtx.err；
* 处理 cancelCtx 的 channel，若 channel 此前未初始化，则直接注入一个 closedChan，否则关闭该 channel；
* 遍历当前 cancelCtx 的 children set，依次将 children context 都进行 cancel；
* 解锁.
* 根据传入的 removeFromParent flag 判断是否需要手动把 cancelCtx 从 parent 的 children set 中移除.

removeChild 方法中，观察如何将 cancelCtx 从 parent 的 children set 中移除：
```go
func removeChild(parent Context, child canceler) {
    p, ok := parentCancelCtx(parent)
    if !ok {
        return
    }
    p.mu.Lock()
    if p.children != nil {
        delete(p.children, child)
    }
    p.mu.Unlock()
}
```
* 如果 parent 不是 cancelCtx，直接返回（因为只有 cancelCtx 才有 children set） 
* 加锁；
* 从 parent 的 children set 中删除对应 child
* 解锁返回.

## timerCtx类
{{< figure src="/images/context06.png" title="go-work (figure)" >}}

```go
type timerCtx struct {
    cancelCtx
    timer *time.Timer // Under cancelCtx.mu.

    deadline time.Time
}
```

timerCtx 在 cancelCtx 基础上又做了一层封装，除了继承 cancelCtx 的能力之外，新增了一个 time.Timer 用于定时终止 context；另外新增了一个 deadline 字段用于字段 timerCtx 的过期时间

## valueCtx
{{< figure src="/images/context07.png" title="go-work (figure)" >}}
```go
type valueCtx struct {
    Context
    key, val any
}
```

* valueCtx 同样继承了一个 parent context；
* 一个 valueCtx 中仅有一组 kv 对

### valueCtx.Value()
{{< figure src="/images/context08.png" title="go-work (figure)" >}}
```go
func (c *valueCtx) Value(key any) any {
    if c.key == key {
        return c.val
    }
    return value(c.Context, key)
}
```

* 假如当前 valueCtx 的 key 等于用户传入的 key，则直接返回其 value；
* 假如不等，则从 parent context 中依次向上寻找.

```go
func value(c Context, key any) any {
    for {
        switch ctx := c.(type) {
        case *valueCtx:
            if key == ctx.key {
                return ctx.val
            }
            c = ctx.Context
        case *cancelCtx:
            if key == &cancelCtxKey {
                return c
            }
            c = ctx.Context
        case *timerCtx:
            if key == &cancelCtxKey {
                return &ctx.cancelCtx
            }
            c = ctx.Context
        case *emptyCtx:
            return nil
        default:
            return c.Value(key)
        }
    }
}
```

* 启动一个 for 循环，由下而上，由子及父，依次对 key 进行匹配；
* 其中 cancelCtx、timerCtx、emptyCtx 类型会有特殊的处理方式；
* 找到匹配的 key，则将该组 value 进行返回


