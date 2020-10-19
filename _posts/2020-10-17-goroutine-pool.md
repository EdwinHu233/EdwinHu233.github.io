---
layout: post
title:  "如何实现一个 Goroutine 池"
---

虽然 Golang 中创建 goroutine 的代价很小，
但似乎还是不能忽视的。
这篇博客就来尝试实现一个 goroutine 池吧。

## 思路

思路其实是很简单的两步：
1. 如何把 goroutine 包装成对象（实现一个 Worker 类）
2. 如何用一个“池”把这些对象管理起来（实现一个 Pool 类）

## 实现

直接放码吧：

```go
type Worker struct {
	jobRequest chan func()
}

func MakeWorker(p *Pool) *Worker {
	w := &Worker{
		jobRequest: make(chan func()),
	}
	go func() {
		for {
			select {
			case <-p.kill:
				return
			case job := <-w.jobRequest:
				job()
				p.mu.Lock()
				p.idleWorkers = append(p.idleWorkers, w)
				p.cond.Signal()
				p.mu.Unlock()
			}
		}
	}()
	return w
}

type Pool struct {
	mu          sync.Mutex
	cond        *sync.Cond
	capacity    int
	size        int
	idleWorkers []*Worker
	waitingJobs []func()
	kill        chan struct{}
}

func (p *Pool) Submit(f func()) {
	p.mu.Lock()
	defer p.mu.Unlock()

	p.waitingJobs = append(p.waitingJobs, f)
	p.cond.Signal()
}

func (p *Pool) canAssignJob() bool {
	return len(p.waitingJobs) > 0 &&
		(len(p.idleWorkers) > 0 || p.size < p.capacity)
}

func (p *Pool) Kill() {
	close(p.kill)
	p.cond.Signal()
}

func MakePool(capacity int) *Pool {
	p := &Pool{
		capacity: capacity,
		kill:     make(chan struct{}),
	}
	p.cond = sync.NewCond(&p.mu)

	go func() {
		for {
			p.mu.Lock()
			for !p.canAssignJob() {
				p.cond.Wait()
			}

			select {
			case <-p.kill:
				p.mu.Unlock()
				return
			default:
			}

			// for p.canAssignJob() {
			job := p.waitingJobs[0]
			p.waitingJobs = p.waitingJobs[1:]

			var w *Worker
			if len(p.idleWorkers) == 0 {
				p.size++
				w = MakeWorker(p)
			} else {
				w = p.idleWorkers[0]
				p.idleWorkers = p.idleWorkers[1:]
			}
			w.jobRequest <- job
			// }

			p.mu.Unlock()
		}
	}()

	return p
}
```

主要逻辑还是比较简单的：
- Pool 会创建最多 capacity + 1 个 goroutine ，其中 capacity 个用来干活，1 个在后台分配任务。
- Pool 负责任务开始的逻辑，Worker 负责任务结束的逻辑。
- 当 Pool 收到新任务时，不会立即让它开始，而是先存在一个队列中。
- 如果队列非空，且 Pool 还没有满（存在闲置的 Worker，或是可以创建新的 Worker），则后台的 goroutine 从队列取出任务分配给 Worker 。
- 有两种情况都会使 `canAssignJob` 由假变为真：原本为空的队列变得不空了，或是某个 Worker 变为闲置状态。对于这两种情况，都要调用 `Signal` ，使得负责后台管理的 goroutine 取消阻塞，并分配任务。
- 用一个 channel `kill` 表示整个线程池是否停止工作。

## 测试

我们做一下对比，看看 Pool 的性能和无限制创建 goroutine 有什么差距：

```go
var numWorkers int = 1 << 10
var numJobs int = 1 << 20

func sleepDuration() time.Duration {
	return time.Millisecond * 1
}

func jobGen(wg *sync.WaitGroup, i int) func() {
	return func() {
		defer wg.Done()
		time.Sleep(sleepDuration())
	}
}

func BenchmarkNoPool(b *testing.B) {
	for caseId := 0; caseId < b.N; caseId++ {
		var wg sync.WaitGroup
		wg.Add(numJobs)

		for i := 0; i < numJobs; i++ {
			job := jobGen(&wg, i)
			go job()
		}

		wg.Wait()
	}
}

func BenchmarkPool(b *testing.B) {
	for caseId := 0; caseId < b.N; caseId++ {
		var wg sync.WaitGroup
		wg.Add(numJobs)
		p := MakePool(numWorkers)

		for i := 0; i < numJobs; i++ {
			job := jobGen(&wg, i)
			p.Submit(job)
		}

		wg.Wait()
		p.Kill()
	}
}
```

结果为：

```
jingbo@harbor ~/c/o/go> go test -bench=. -benchtime=10x
goos: linux
goarch: amd64
pkg: other
BenchmarkNoPool-16    	      10	 667311250 ns/op
BenchmarkPool-16      	      10	2131090776 ns/op
PASS
ok  	other	31.006s
```

~~emmmm 可以看到，用 Pool 的时间代价比无限制创建 goroutine 要高出很多。大概这段代码还有很大的优化空间。不过能用一千个 goroutine 去解决一百万个并发任务，还算可以接受吧（？~~

我又想了想，对于 CPU bound 的任务，这种 goroutine 池还是很有必要的。

具体来说，越是偏向 CPU bound ，就越应该限制总并发数（极限情况是总并发数 == 可用来并发的物理 CPU 核心数），
因为在这种情况下，无限制的并发并不会加快计算，反而会增加 goroutine 上下文切换的总代价，
白白浪费有限的 CPU 资源。

此外，对于需要获取其他有限资源的任务，这种 goroutine 池也是有意义的。
例如操作系统对一个进程能够同时打开的文件数是有限制的。
在 Linux 中，我们可以通过 `ulimit` 指令查看这个限制数：

```
jingbo@harbor ~> ulimit -aH
Maximum size of core files created                           (kB, -c) unlimited
Maximum size of a process’s data segment                     (kB, -d) unlimited
Maximum size of files created by the shell                   (kB, -f) unlimited
Maximum size that may be locked into memory                  (kB, -l) 64
Maximum resident set size                                    (kB, -m) unlimited
Maximum number of open file descriptors                          (-n) 524288
Maximum stack size                                           (kB, -s) unlimited
Maximum amount of cpu time in seconds                   (seconds, -t) unlimited
Maximum number of processes available to a single user           (-u) 61396
Maximum amount of virtual memory available to the shell      (kB, -v) unlimited
```

如果无限制地创建 goroutine ，并且每个 goroutine 都要打开一个新文件的话，
会迅速达到操作系统设置的上限。
