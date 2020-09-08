---
layout: post
title:  "Golang 笔记：select 的几种用法"
---

* TOC
{:toc}

> 这篇博客是 The Go Programming Language 第 8.7 节的笔记

Golang 中的 `select` 语句可以从多个 channel 读写操作中选择一个非阻塞操作来执行
（如果有多个非阻塞的，则等概率选择其中一个）。
整体结构与 `switch` 相似，由多个分支构成，
每个分支是 channel 上的一个读写操作。
`select` 语句还可以有一个可选的 `default` 分支，
会在所有 channel 读写操作均阻塞时被执行。

下面介绍 `select` 的几种典型用法。

## 超时机制

如果某一种或几种事件在一段时间内未发生，
则进行特定的操作。
这种超时机制可以用 `select` 和 `time.After` 搭配实现。

### time.After
官方文档对于 `time.After` 的描述如下：

```
func After(d Duration) <-chan Time

After waits for the duration to elapse and then sends the current time
on the returned channel. It is equivalent to NewTimer(d).C. The
underlying Timer is not recovered by the garbage collector until the
timer fires. If efficiency is a concern, use NewTimer instead and call
Timer.Stop if the timer is no longer needed.
```

可知， `time.After` 返回的 channel 与一个 `time.Timer` 对象关联
（准确来说，返回的 channel 是 `time.Timer` 对象的一个字段）。
在经过一段时间后，这个 channel 上会出现当前的时间值。

不过，与这个 channel 关联的 `time.Timer` 对象不会被 GC 回收。
即使 channel 已经不可用，也要等到时间结束，相关资源才会被释放（例如 goroutine ）。

`time.After` 的用例如下（一个模拟火箭发射倒计时的程序）：

```go
func main() {
    abort := make(chan bool)
    fmt.Println("Starting countdown. Press return to abort")
    go func() {
        reader := bufio.NewReader(os.Stdin)
        for {
            b, err := reader.ReadByte()
            if err != nil {
                log.Printf("When reading from stdin: %v", err)
                return
            }
            if b == '\n' {
                abort <- true
            }
        }
    }()

    select {
    case <-time.After(10 * time.Second):
        // do nothing
    case <-abort:
        fmt.Println("Launch aborted")
        return
    }
    fmt.Println("Launch!")
}
```

### time.Timer

要实现更灵活的定时机制，需要用到 `time.Timer` 。
官方文档对于 `time.Timer` 的描述如下：

```
The Timer type represents a single event. When the Timer expires, the current
time will be sent on C, unless the Timer was created by AfterFunc. A Timer
must be created with NewTimer or AfterFunc.

type Timer struct {
    C <-chan Time
    // contains filtered or unexported fields
}


func AfterFunc(d Duration, f func()) *Timer

AfterFunc waits for the duration to elapse and then calls f in its own
goroutine. It returns a Timer that can be used to cancel the call using
its Stop method.


func NewTimer(d Duration) *Timer

NewTimer creates a new Timer that will send the current time on its channel
after at least duration d.
```

可知，`time.Timer` 对象必须由 `time.AfterFunc` 或 `time.NewTimer` 创建。
前者会在等待一段时间后执行特定的函数（可以用 `Stop` 方法中止），
后者会在等待一段时间后在 channel 上发送此时的时间值（也可以用 `Stop` 方法中止）。

`time.After` 的用法在逻辑上近似于后者。

## 定时机制

每隔一段时间就进行特定的操作。
这种定时机制可以用 `select` 与 `time.Tick` 或 `time.NewTicker` 配合实现。

### time.Tick

官方文档对于 `time.Tick` 的描述如下：

```
func Tick(d Duration) <-chan Time

Tick is a convenience wrapper for NewTicker providing access to the
ticking channel only. While Tick is useful for clients that have no
need to shut down the Ticker, be aware that without a way to shut it
down the underlying Ticker cannot be recovered by the garbage collector;
it "leaks". Unlike NewTicker, Tick will return nil if d <= 0.
```

可知，`time.Tick` 返回的 channel 与一个 `time.Ticker` 对象关联
（准确来说，返回的 channel 是这个对象的字段）。
这个 channel 上每隔一段时间就会有当前的时间值。

不过，与这个 channel 关联的 `time.Ticker` 无法被 GC 回收，
也无法手动停止。
因此，这个函数只适合在 main 函数中使用，否则会发生 goroutine leak 。

`time.Tick` 的用例如下：

```go
func main() {
    abort := make(chan bool)
    fmt.Println("Starting countdown. Press return to abort")
    go func() {
        reader := bufio.NewReader(os.Stdin)
        for {
            b, err := reader.ReadByte()
            if err != nil {
                log.Printf("When reading from stdin: %v", err)
                return
            }
            if b == '\n' {
                abort <- true
            }
        }
    }()

    tick := time.Tick(1 * time.Second)
    for i := 10; i > 0; i-- {
        select {
        case <-tick:
            fmt.Println(i)
        case <-abort:
            fmt.Println("Abort launching.")
            return
        }
    }
    fmt.Println("Launch!")
}
```

### time.NewTicker

为了更灵活地实现定时机制，
我们可以用 `time.NewTicker` 函数创建`time.Ticker` 对象。

官方文档的描述如下：

```
A Ticker holds a channel that delivers `ticks' of a clock at
intervals.

type Ticker struct {
    C <-chan Time // The channel on which the ticks are delivered.
    // contains filtered or unexported fields
}


func NewTicker(d Duration) *Ticker

NewTicker returns a new Ticker containing a channel that will send
the time with a period specified by the duration argument. It adjusts
the intervals or drops ticks to make up for slow receivers. The
duration d must be greater than zero; if not, NewTicker will panic.
Stop the ticker to release associated resources.
```

可知，`time.Ticker` 的计时功能可以用 `Stop` 手动停止；
停止后所有相关资源都会被释放（例如 goroutine ）。

## 非阻塞通信

由于 `select` 可以有 `default` 分支，
所以可以用来实现非阻塞通信。

```go
select {
case <-ch:
    // recieved communication
default:
    // haven't recieved (non-blocking)
}
```

注意不要与 `x, ok := <-ch` 的写法混淆：
一个是判断读写操作是否会阻塞，另一个是判断 channel 是否已关闭。

## nil channel

channel 的零值为 `nil` ，对一个 nil channel 进行读写会导致永久阻塞。
因此，当 `select` 的某一分支中的 channel 为 `nil` 时，
这个分支永远不会被选中。
如果想开启或关闭程序的某些 feature ，可以使用这个特性。
