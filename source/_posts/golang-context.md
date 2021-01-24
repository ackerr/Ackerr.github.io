---
title: 深入浅出Golang之Context 
date: 2021-01-24 22:26:09
tags:
    - go
---

通过`context.Context` ，我们可以在 goruntine 中传递上下文参数以及同步取消信号。例如在处理http请求或记录请求链路时，可通过 Context 在goruntine间传递信息。也可以通过Context的超时取消，实现[graceful shutdown](https://wzmmmmj.com/2020/10/31/golang-gracefully-upgrade/)。本文从源码角度，分析`context.Context`是如何实现上下文传递以及同步取消信号。

> 后遗症：具有网络请求的第三方库，如果无法传递Context，都不太想用，因为路径追踪中无法显示。

## 接口实现

`context.Context`接口

```go
type Context interface {
	Deadline() (deadline time.Time, ok bool)
	Done() <-chan struct{}
	Err() error
	Value(key interface{}) interface{}
}
```

- Deadline：返回Context被取消的截止时间，如果第二个参数返回fasle，则说明未设置截止时间；
- Done：返回一个channel对象，Context被取消后，该channel会被close；
- Err：返回Context结束原因。当Done返回的channel未被取消时，返回nil，被取消则返回取消原因。例如Context因为超时关闭，返回`DeadlineExceeded`；
- Value：返回Context中保存的键值；

`context.canceler`接口

```go
type canceler interface {
	cancel(removeFromParent bool, err error)
	Done() <-chan struct{}
}
```

- cancel：取消当前Context;
- Done：与Context接口一致，返回一个channel对象

## 基本原理

context包中提供了两个常用方法，`context.Background`和`context.TODO`。

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

从源码上看，两者并没什么不同，均通过`new(emptyCtx)`初始化。使用场景上，`context.Backgound`常用于主函数、初始化、以及测试用例中，作为顶层上下文进行传递，仅当不确定使用哪种Context时，才通过`context.TODO`占坑。对应emptyCtx实现如下：

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

func (*emptyCtx) Value(key interface{}) interface{} {
	return nil
}
```

不难发现，这两个 Context 啥都没干，没有截止时间，不能被取消，也不能设置上下文参数，仅仅通过空方法实现了Context接口。如果需要实现传递上下文、同步取消信号等额外功能，可通过context包中提供的方法进行拓展，当然也可以自己实现Context接口。

## WithValue

`context.WithValue`基于`parent Context`，生成valueCtx类型Context，并保留一对键值，常用来传递上下文。

```go
func WithValue(parent Context, key, val interface{}) Context {
	if parent == nil {
		panic("cannot create context from nil parent")
	}
	if key == nil {
		panic("nil key")
	}
    // 因为传递的是interface，这里通过反射，判断key类型能否进行比较
	if !reflectlite.TypeOf(key).Comparable() {
		panic("key is not comparable")
	}
	return &valueCtx{parent, key, val}
}

type valueCtx struct {
	Context
	key, val interface{}
}

func (c *valueCtx) Value(key interface{}) interface{} {
	// 比较key是否等于当前context key，如果不是则向父context查询
    if c.key == key {
		return c.val
	}
	return c.Context.Value(key)
}
```

需要注意的是，`valueCtx.Value`实现了链式查找，如果当前context中为找到为符合的key值，则会向父context中继续。

### 举个例子

示例代码中，ctx获取到grandparent context的value。

```go
package main

import "context"

func main() {
	ctx := context.Background()
	ctx = context.WithValue(ctx, "key1", "1")
	ctx = context.WithValue(ctx, "key2", "2")
	ctx = context.WithValue(ctx, "key3", "3")
	println(ctx.Value("key1").(string))  // "1"
}
```

## WithCancel

`context.WithCancel`常用于控制派生 goruntine，通过接收`parent Context`，返回两个参数`ctx cancelCtx`和`cancel cancelFunc`。

```go
type CancelFunc func()

func WithCancel(parent Context) (ctx Context, cancel CancelFunc) {
	if parent == nil {
		panic("cannot create context from nil parent")
	}
    // 通过parent生成一个cancelCtx，会设置一个新的done channel对象
	c := newCancelCtx(parent)
    // 与parent关联
	propagateCancel(parent, &c)
	return &c, func() { c.cancel(true, Canceled) }
}

func newCancelCtx(parent Context) cancelCtx {
	return cancelCtx{Context: parent}
}

type cancelCtx struct {
	Context

	mu       sync.Mutex
	done     chan struct{}
	children map[canceler]struct{}
	err      error
}
```

这里主要看下`propagateCancel`的实现。

```go
// parent 父上下文
// child  当前上下文
func propagateCancel(parent Context, child canceler) {
	done := parent.Done()
    // parent不会触发cancel，直接返回
    // 例如context.Background()这种Done方法直接返回nil的
	if done == nil {
		return
	}

	select {
    // parent已经被cancel，child直接执行cancel
	case <-done:
		child.cancel(false, parent.Err())
		return
	default:
	}
    // 判断parent是否是cancelCtx类型
	if p, ok := parentCancelCtx(parent); ok {
        // parent是cancelCtx类型
		p.mu.Lock()
		if p.err != nil {
			// 该ctx已经被cancel
			child.cancel(false, p.err)
		} else {
            // 将当前context加入到上层cancelCtx的children中
			if p.children == nil {
				p.children = make(map[canceler]struct{})
			}
			p.children[child] = struct{}{}
		}
		p.mu.Unlock()
	} else {
        // parent不是cancelCtx类型，但可被cancel，监听parent和child是否被取消
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

通过源码，`propagateCancel`函数主要作用就是关联当前context和父context，如果父context是cancelCtx类型并且未被cancel，则加入到它的children字段中。当执行 cancel 函数时，除了关闭自身`done channel`外，还会为 children 中关联的所有context执行 cancel 方法。通过Err方法，可获取`context.Canceled ` 错误。`cancelCtx.cancel`实现如下

```go
var Canceled = errors.New("context canceled")

// removeFromParent 是否需要从parent.children中移除
func (c *cancelCtx) cancel(removeFromParent bool, err error) {
	if err == nil {
		panic("context: internal error: missing cancel error")
	}
	c.mu.Lock()
    // err不为nil，说明已经被取消，直接返回
	if c.err != nil {
		c.mu.Unlock()
		return
	}
    // 设置err，并关闭done chan
	c.err = err
	if c.done == nil {
		c.done = closedchan
	} else {
		close(c.done)
	}
    // 关联的所有子context依次执行cancel
	for child := range c.children {
		child.cancel(false, err)
	}
	c.children = nil
	c.mu.Unlock()
    
	if removeFromParent {
        // 在parent的children字段中移除该context
		removeChild(c.Context, c)
	}
}
```

### 举个例子

示例代码，sleep 3秒后，因为执行 cancel，ctx.Err()输出`context canceled`。

```go
package main

import (
	"context"
	"fmt"
	"time"
)

func main() {
	ctx := context.Background()
	ctx, cancel := context.WithCancel(ctx)
	go f(ctx)
	time.Sleep(time.Second * 3)
	cancel()
	time.Sleep(time.Second * 1)
}

func f(ctx context.Context) {
	i := 1
	for {
		select {
		case <-ctx.Done():
			fmt.Println(ctx.Err())
			return
		default:
			fmt.Println(i)
			time.Sleep(time.Second * 1)
			i++
		}
	}
}
```

## WithDeadline

`context.WithDeadline`常用于需要超时关闭的场景。通过设置截止时间，起到超时自动取消context以及子context的效果。当context因为超时被 cancel 时，通过Err方法，可获取`context.DeadlineExceeded` 错误，代码如下

```go
func WithDeadline(parent Context, d time.Time) (Context, CancelFunc) {
	if parent == nil {
		panic("cannot create context from nil parent")
	}
    // 判断parent是否设置截止时间，并且是否达到该时间
	if cur, ok := parent.Deadline(); ok && cur.Before(d) {
        // 如果d时间已过，则返回cancelCtx类型context，截止时间使用parent的
		return WithCancel(parent)
	}
    // 将父级context先包装成cancelCtx类型
	c := &timerCtx{
		cancelCtx: newCancelCtx(parent),
		deadline:  d,
	}
    // 关联parent
	propagateCancel(parent, c)
    // 如果当前时间已经超过了截止时间，执行cancel，直接返回一个已经被 cancel 的 timerCtx
	dur := time.Until(d)
	if dur <= 0 {
		c.cancel(true, DeadlineExceeded)
		return c, func() { c.cancel(false, Canceled) }
	}
	c.mu.Lock()
	defer c.mu.Unlock()
	if c.err == nil {
        // 启动一个定时器，到截止时间自动取消这个 timerCtx
		c.timer = time.AfterFunc(dur, func() {
			c.cancel(true, DeadlineExceeded)
		})
	}
	return c, func() { c.cancel(true, Canceled) }
}
```

### 举个例子

示例代码，执行3秒后，定时器执行 cancel，ctx.Err()输出`context deadline exceeded`。

```go
package main

import (
	"context"
	"fmt"
	"time"
)

func main() {
	ctx := context.Background()
    ctx, cancel := context.WithDeadline(ctx, time.Now().Add(time.Second * 3))
	defer cancel()
	go f(ctx)
	time.Sleep(time.Second * 5)
}

func f(ctx context.Context) {
	i := 1
	for {
		select {
		case <-ctx.Done():
			fmt.Println(ctx.Err())
			return
		default:
			fmt.Println(i)
			time.Sleep(time.Second * 1)
			i++
		}
	}
}
```

## WithTimeout

与`context.WithDeadline`函数类似，只不过传参从time.Time类型该为time.Duration，也就是截止时间改为超时时间。实际调用的其实还是`context.WithDeadline`

```go
func WithTimeout(parent Context, timeout time.Duration) (Context, CancelFunc) {
	return WithDeadline(parent, time.Now().Add(timeout))
}
```
