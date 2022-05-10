---
layout: post
title: Go利器-Context学习
categories: Golang
---
### 1. 背景

context.Context常用于多个goroutine之间同步取消、截止时间和携带额外信息。

### 2. 接口设计

#### Context interface

```go
type Context interface {
    Deadline() (deadline time.Time, ok bool)
    Done() <-chan struct{}
    Err() error
    Value(key interface{}) interface{}
}
```

其中包括四个接口：

- Deadline 返回截止时间，如未设置截止时间，则返回ok=false
- Done 返回一个channel，它在工作完成时或者上下文取消时被关闭；如ctx不会被取消，则返回nil。
- Err 返回ctx取消的原因，常见两个DeadlineExceeded 和 Canceled
- Value 携带的特定数据，从valueContext中取出，若当前ctx不存在，则从parentCtx中查找

#### 常用接口

- Background  内部实现是一个emptyCtx，不可取消，没有截止时间，不可携带数据。
- TODO 同上，适用场景有稍微区别，仅在不确认使用哪种上下文时使用。
- WithCancel  从parentContext衍生一个新的子上下文，并返回用于取消该context的函数。当取消函数执行时，该context以及树形结构上的所有衍生的子context都会被取消。此时Done函数返回的channel被关闭，以及Err函数会返回context.Canceled错误码。
- WithDeadline 创建一个timeCtx，通过time.AfterFunc实现超时逻辑，当时间超过截止时间后，调用timeCtx.cancel来同步取消。
- WithTimeout 内部调用withDealline，创建超时context.timeCtx.
- WithValue 用于创建一个valueCtx实例，携带额外的信息；查询时通过valueCtx.Value方法查询，如果key,value不匹配，则从父ctx中继续查找，直到返回对应的值或者nil。

### 3. 使用场景

#### 3.1 RPC链路超时

分别是rpc client端和server端的两个例子

```go
// rpcInvoke client端发起rpc请求，timeout 自定义的超时时间
func rpcInvoke(ctx context.Context, timeout int32) error {
    // 如果携带超时
    ctx, cancel := context.WithTimeout(ctx, time.Duration(timeout)*time.Millisecond)
    defer cancel()
    // udp 发包
    srcAddr := &net.UDPAddr{IP: net.IPv4zero, Port: 0}
    dstAddr := &net.UDPAddr{IP: net.IPv4zero, Port: 8080}
    conn, err := net.DialUDP("udp", srcAddr, dstAddr)
    if err != nil {
        return err
    }
    // 判断是否超时
    if ctx.Err() == context.DeadlineExceeded {
        return fmt.Errorf("dial timeout")
    }
    if _, err = conn.Write([]byte("hello")); err != nil {
        return err
    }
    // ....TODO read
    return nil
}

// rpcHandle server处理请求，上游携带了超时时间timeout
func rpcHandle(ctx context.Context, timeout int32) error {
    // 如果携带超时
    ctx, cancel := context.WithTimeout(ctx, time.Duration(timeout)*time.Millisecond)
    defer cancel()
    // 异步处理逻辑，必须在规定时间内处理结束
    response := make(chan int32)
    go func() {
        // 业务逻辑需要3s才能处理完
        time.Sleep(time.Second*3)
        response <- 1
    }()
    select {
        // 等待超时
        case <- ctx.Done():
            fmt.Printf("context err: %v\n", ctx.Err())
            return fmt.Errorf("handle err: %v", ctx.Err())
        case r := <- response:
            fmt.Printf("handle success: ret: %d\n", r)
    }
    return nil
}
```

#### 3.2 RPC ctx参数携带

```go
const ContextTraceKey = "TraceKey"

func WithTraceID(ctx context.Context, traceID string) context.Context {
    return context.WithValue(ctx, ContextTraceKey, traceID)
}

func TraceID(ctx context.Context) string {
    id, ok := ctx.Value(ContextTraceKey).(string)
    if ok {
        return id
    }
    return ""
}
```

### 4.源码分析

#### 4.1 可撤销ctx实现

```go
func propagateCancel(parent Context, child canceler) {
    // 父context不会触发cancel信号，直接return
    done := parent.Done()
    if done == nil {
        return // parent is never canceled
    }
    // 判断父context是否已经被取消了
    select {
    case <-done:
        // parent is already canceled
        child.cancel(false, parent.Err())
        return
    default:
    }
    // 向上查找可被cancel的父context
    if p, ok := parentCancelCtx(parent); ok {
        p.mu.Lock()
        if p.err != nil {
            // parent has already been canceled
            // 已经被cancel
            child.cancel(false, p.err)
        } else {
            // 放入p的子context中，等待父context的信号
            if p.children == nil {
                p.children = make(map[canceler]struct{})
            }
            p.children[child] = struct{}{}
        }
        p.mu.Unlock()
    } else {
        // 没有找到可被cancel的父context，单独启动协程监听parent和child信号
        // 还有一种case：自定义的Context类型？
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

- 父ctx撤销时，会向所有子ctx传递撤销信号，此时是深度优先的。

#### 4.2 Value ctx实现

在Context的基础上，附加了一个key, value interface{} 用于携带信息

```go
type valueCtx struct {
    Context
    key, val interface{}
}
```

然后，在查找的时候，先判断当前ctx有无数据，如无，则向上递归查找

```go
func (c *valueCtx) Value(key interface{}) interface{} {
    if c.key == key {
        return c.val
    }
    return c.Context.Value(key)
}
```
