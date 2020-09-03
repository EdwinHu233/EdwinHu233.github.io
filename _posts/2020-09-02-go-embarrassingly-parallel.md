---
layout: post
title: "Golang 笔记：“尴尬的并行”"
---

> 这篇博客是 *The Go Programming Language* 8.5 节的笔记。

有一类简单的并行问题：
所有输入数据两两之间都是独立的，
我们只需要对每个输入数据分别求解，
最后把结果汇总起来。
这种问题被称为 "embarrassingly parallel" 。

下面考虑几种不同的情况。

## 事先知道输入数据的个数

假设我们不需要返回值的话，
可以这样处理：

{% highlight go %}
// 情况 1
// 预先知道输入的个数，且不需要回收结果

func foo1(i int) {
}

func parallel1(ns []int) {
    ch := make(chan struct{}) 
    for _, n := range ns {
        go func(n int) {
            log.Printf("enter foo1: %d\n", n)
            defer log.Printf("exit foo1: %d\n", n)
            foo1(n)
            ch <- struct{}{}
        }(n)
    }
    for range ns {
        <-ch
    }
}
{% endhighlight %}

如果我们需要回收所有子问题的返回值的话，
只需要对上面的代码稍加修改即可。
但是，如果在回收返回值的过程中可能半途而废
（例如，返回值是 `error` 类型且不为 `nil` ），
这种思路的代码就会出错。

{% highlight go %}
// 情况 2
// 预先知道输入的个数，需要回收结果，
// 且可能在回收过程中半途而废

func foo2(i int) error {
    if i%2 == 0 {
        return fmt.Errorf("foo: even")
    }
    return nil
}

// 错误做法

func parallel2_bad(ns []int) error {
    ch := make(chan error) // unbuffered channel
    for _, n := range ns {
        go func(n int) {
            log.Printf("enter foo2: %d\n", n)
            defer log.Printf("exit foo2: %d\n", n)
            ch <- foo2(n)
        }(n)
    }
    for range ns {
        if err := <-ch; err != nil {
            return err // goroutine leak!
        }
    }
    return nil
}
{% endhighlight %}

错误的原因在于，
`ch` 是一个 unbuffered channel 。
因此在 `return err` 的时候，
`ch` 中仍有元素未取出，
就会导致后续想要写入 `ch` 的 goroutine 永久阻塞。
这种情况就称为 "goroutine leak" 。

要修复这个错误，
最简单的方法就是把 `ch` 初始化为 buffered channel ：

{% highlight go %}
// 正确做法

func parallel2_good(ns []int) error {
    ch := make(chan error, len(ns)) // buffered channel
    for _, n := range ns {
        go func(n int) {
            log.Printf("enter foo2: %d\n", n)
            defer log.Printf("exit foo2: %d\n", n)
            ch <- foo2(n)
        }(n)
    }
    for range ns {
        if err := <-ch; err != nil {
            return err
        }
    }
    return nil
}
{% endhighlight %}

## 事先不知道输入数据的个数

对于这种情况，
我们在逻辑上需要一个计数器：
当创建 goroutine 时，计数器 +1；
当 goroutine 结束时，计数器 -1 。
Golang 的标准库中已经内置了这样的“计数器”类型，
就是 `sync.WaitGroup`：

{% highlight go %}
// 情况 3
// 事先不知道输入数据的个数，需要回收数据
// 且不会半途而废

func foo3(n int) int {
    return n * n
}

func parallel3(ns chan int) int {
    ch := make(chan int)
    var wg sync.WaitGroup
    for n := range ns {
        wg.Add(1)
        go func(n int) {
            log.Printf("enter foo3: %d\n", n)
            defer log.Printf("exit foo3: %d\n", n)
            defer wg.Done()
            ch <- foo3(n)
        }(n)
    }
    go func() {
        wg.Wait()
        close(ch)
    }()
    var sum int
    for result := range ch {
        sum += result
    }
    return sum
}
{% endhighlight %}

这段代码要注意几个细节。

1. `wg.Add(1)` 和 `wg.Done()` 的非对称性。
    - 很容易理解，后者应该放在 goroutine 的结尾；
    但为什么要把前者放在 goroutine 之前呢？
    这是为了保证 `wg.Add(1)` 一定发生在后面的 `wg.Wait()` 之前。

2. `wg.Wait()` 和 `close(ch)` 被放在了一个 "closer goroutine" 中，与后面的 for 循环并行
    - 在逻辑上，当所有 goroutine 结束后，就应该关闭 `ch` 。
    所以我们要把 `wg.Wait()` 和 `close(ch)` 放在一起。
    - 如果这两句话在 for 循环之前顺序执行的话，
    主 goroutine 会首先在 `wg.Wait()` 处阻塞，
    无法从 `ch` 中取出元素。
    在 `ch` 被放入第一个元素后，
    后续试图向 `ch` 中放置元素的 goroutine 会全部阻塞，
    导致 `defer wg.Done()` 无法执行。
    所以主 goroutine 和子 goroutine 都会永久阻塞
    （除了第一个向 `ch` 放入元素的子 goroutine ）。
    - 如果这两句话在 for 循环之后顺序执行的话，
    由于 `ch` 未能关闭，主 goroutine 会在 for 循环处永久阻塞。
