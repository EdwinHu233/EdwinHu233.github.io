---
layout: post
title:  "读书笔记：Concurrency in Go"
---

* TOC
{:toc}

> 本文是 Concurrency in Go 的读书笔记。

## An Introduction to Concurrency

大概讲了一下 concurrency 有什么困难（race condition, atomicity, deadlocks, livelocks, starvation）。

## Modeling Your Code: Communicating Sequential Processes

讲了一下 Golang 推崇的并发模型（CSP）与普通的 memory access synchronization 有什么区别，以及如何选择：

![](/asset/concurrency-in-go/ch2-decision-tree.png)

- 是否要转移数据的所有权
    - 如果是的话，用 CSP 比较好（第三章有讲 channel 的一些使用原则）
- 是否要保护某个 struct 的内部结构
    - 如果是的话，用 memory access synchronization 比较好。比如在方法内部加 mutex
    - 一般应该确保 mutex 只在 **内部** 使用。如果暴露在外界的话，可能是抽象层次有问题
- 是否要协调多个部分的逻辑
    - 如果是的话，用 CSP 更好。因为 channel 更容易做到 composition
- critical section 是否对性能有足够的影响
    - 如果是的话，*可能* 用 memory access synchronization 比较好。因为 channel 内部还是要用 memory access synchronization primitives 的，性能上 *可能* 不及直接用 primitives

## Go’s Concurrency Building Blocks

### Goroutines

给了几个简单的例子。此外，还做了几个 benchmark ，说明了 goroutine 的轻量级：
- 空间上，创建一个新的 goroutine 只需要几 kb 的内存开销（44页）。在一台内存 8GB 的个人电脑上，大概能开上百万个 goroutine
- 时间上，goroutine 的上下文切换发生在用户态，时间开销是普通 thread 的 1/10 左右（46页）

最后的结论是：

> Creating goroutines is very cheap, and so you should only be discussing their cost if you’ve proven they are the root cause of a performance issue.

### The sync Package

#### WaitGroup

WaitGroup 的使用场景是，创建多个 goroutine 后，需要等待它们全部结束后再继续执行。
{% comment %} 并且，这些新的 goroutine 要么返回的结果不重要，要么有其他回收结果的方法（而不是直接通过函数返回值）。 {% endcomment %}

WaitGroup 在逻辑上可以看作一个并发安全的计数器，初始值为 0 。
在创建新 goroutine 时，调用 `Add(n)` 方法使计数值增加；
在每个 goroutine 结束时（或者 `defer`）调用 `Done` 使计数值减 1 ；
在 parent goroutine 中调用 `Wait` 方法，阻塞直到计数值重新变为 0 。

如果作为函数参数的话，应该使用指针类型 `*sync.WaitGroup` 。

#### Mutex and RWMutex

Mutex 和 RWMutex 都有 `Lock` 和 `Unlock` 方法，
且作用相同（都会阻塞后续的读者/写者）。

此外 RWMutex 还有 `RLock` 和 `RUnlock` 方法，
可以满足多个同时的读者，或是一个写者。

sync package 中还定义了一个接口 Locker ，
这个接口要求 `Lock` 和 `Unlock` 方法。
Mutex 和 RWMutex 都满足此接口。

此外，RWMutex 有 `RLocker` 方法，
其返回值的类型也有 `Lock` 和 `Unlock` 方法，
但实际调用的是 `RLock` 和 `RUnlock` 。

如果是用 Mutex 或 RWMutex 对一个 Locker 类型的变量赋值，
需要使用指针类型 `*sync.Mutex` 或 `*sync.RWMutex` 。
而 `RLocker` 的返回值则可以直接赋值，不必加指针。

#### Cond

官方文档中是这样描述 Cond 的目的：
> ...a rendezvous point for goroutines waiting for or announcing the occurrence of an event.

换言之，在逻辑上可以有若干个 goroutine 在等待某个事件的发生；
当这个事件发生后，可以唤醒一个或多个被阻塞的 goroutine 。

一般使用 `sync.NewCond(l sync.Locker)` 方法创建 Cond 。
例如 `cond := sync.NewCond(&sync.Mutex{})` 。

作为等待事件发生的一方，一般采用这种模式：

```go
c.L.Lock()
for !condition() {
    c.Wait()
}

// critical section

c.L.Unlock()
```

其中 `Wait` 方法需要特别注意：
在开始时，这个方法会调用 `c.L.Unlock` ，并阻塞 goroutine ，
直到其他 goroutine 调用 `Signal` 或 `Broadcast` 方法。
在结束时，这个方法会重新调用 `c.L.Lock` 。

至于为什么要在一个循环内判断条件并执行 `Wait` 方法，
可以考虑两点：
1. `Signal` 或 `Broadcast` 方法只是在逻辑上宣布某个事件的发生，但这个事件并不一定就是当前 goroutine 期待的事件
2. `Wait` 方法内部，在 goroutine 苏醒到重新获取锁的这段时间内，所期待的事件可能失效

因此，在循环结束后，当前 goroutine 必定持有锁，并且它期待的事件为真。

作为导致事件发生的一方，一般采用这种模式：

```go
c.L.Lock()

// critical section

c.L.Unlock()
c.Signal() // or Broadcast()
```

- `Signal` 会唤醒一个被这个 Cond 阻塞的 goroutine （如果有的话）
- `Broadcast` 会唤醒所有被这个 Cond 阻塞的 goroutine （如果有的话）

#### Once

Once 类型只有一个方法 `Do` ，这个方法接受一个 `func()` 类型的函数对象。
简单来说，当 Once 类型的对象第一次被调用 `Do` 时，
会实际执行传入函数对象。
之后再次调用 `Do` 时，则不会实际执行（即使传入的参数不同）。

不过，还是有一个地方需要注意：
当 `Do` 被同时执行多次的话，
其中第一个会实际执行，其他则会阻塞。
因此，如果传入的函数对象也会调用 `Do` 的话，
就会导致 deadlock 。
一个最小化的示例如下：

```go
var once sync.Once
empty := func() {}
once.Do(func() { once.Do(empty) })
```
