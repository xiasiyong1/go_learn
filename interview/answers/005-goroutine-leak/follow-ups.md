# 005. goroutine 泄漏 - 面试追问

## 1. 如何从 goroutine 堆栈判断泄漏卡在哪里？

看 goroutine profile 时，重点看大量 goroutine 的顶部栈帧和状态。

常见状态包括：

- `chan send`：卡在发送，通常没人接收。
- `chan receive`：卡在接收，通常没人发送或 channel 没关闭。
- `select`：要看 select 里有哪些 case，是否缺少 `ctx.Done()`。
- `IO wait`：卡在网络或文件 I/O，可能缺 timeout。
- `semacquire`：卡在锁或 WaitGroup，可能有死锁或等待方永远不结束。

例如看到很多 goroutine 都停在这一行：

```go
out <- value
```

就应该继续问：谁负责接收 `out`？接收方是否提前退出？发送方有没有监听取消信号？

修正通常不是“把 channel 改成 buffered”，而是补上退出协议：

```go
select {
case out <- value:
case <-ctx.Done():
	return
}
```

buffer 只能缓解阻塞，不能证明 goroutine 一定能退出。

## 2. `context` 为什么不能自动杀死 goroutine？

`context` 只是取消信号，不是线程中断机制。goroutine 必须主动检查 `ctx.Done()`。

错误示例：

```go
func worker(ctx context.Context) {
	go func() {
		for {
			doWork() // 永远不看 ctx
		}
	}()
}
```

即使外部调用了 `cancel()`，这个 goroutine 也不会自动退出。

正确写法：

```go
func worker(ctx context.Context) {
	go func() {
		for {
			select {
			case <-ctx.Done():
				return
			default:
				doWork()
			}
		}
	}()
}
```

如果 `doWork()` 本身可能阻塞很久，还要把 context 继续传进去：

```go
func worker(ctx context.Context) {
	go func() {
		for {
			if err := doWork(ctx); err != nil {
				return
			}
		}
	}()
}
```

面试里要把这句话说清楚：取消信号必须传到所有可能阻塞的位置。

## 3. pipeline 下游提前退出时，上游为什么会泄漏？

因为上游通常在发送 channel 上阻塞，等待下游接收。如果下游拿到一个结果就返回，上游就没人接收了。

错误示例：

```go
func numbers() <-chan int {
	out := make(chan int)
	go func() {
		for i := 0; ; i++ {
			out <- i
		}
	}()
	return out
}

func takeOne() int {
	return <-numbers()
}
```

`takeOne` 返回后，`numbers` 里的 goroutine 会继续尝试发送下一个数字，然后永久阻塞。

正确写法是传入取消信号：

```go
func numbers(ctx context.Context) <-chan int {
	out := make(chan int)
	go func() {
		defer close(out)
		for i := 0; ; i++ {
			select {
			case out <- i:
			case <-ctx.Done():
				return
			}
		}
	}()
	return out
}

func takeOne(parent context.Context) int {
	ctx, cancel := context.WithCancel(parent)
	defer cancel()

	return <-numbers(ctx)
}
```

下游退出前取消 context，上游才能从发送阻塞里退出。

## 4. `time.Tick` 和 `time.NewTicker` 在泄漏问题上有什么区别？

`time.Tick` 返回一个 channel，但你拿不到 ticker 对象，也就不能调用 `Stop`。更重要的是，很多人会配合 `for range time.Tick(...)` 写出没有退出条件的后台循环。

不推荐：

```go
func start() {
	go func() {
		for range time.Tick(time.Second) {
			report()
		}
	}()
}
```

推荐：

```go
func start(ctx context.Context) {
	go func() {
		ticker := time.NewTicker(time.Second)
		defer ticker.Stop()

		for {
			select {
			case <-ticker.C:
				report()
			case <-ctx.Done():
				return
			}
		}
	}()
}
```

面试中可以补一句：长期运行服务里，凡是有后台 ticker，通常都要同时有 `Stop` 和退出信号。这样代码不依赖运行时如何回收 timer，而是从业务生命周期上明确可退出。

## 5. 用 goroutine 包一层慢调用，为什么可能让问题更严重？

因为这只是把阻塞转移到了后台。如果没人接收结果，或者慢调用没有超时，goroutine 仍然会泄漏。

错误示例：

```go
func queryWithTimeout(query string) (Result, error) {
	ch := make(chan Result)
	go func() {
		ch <- slowQuery(query)
	}()

	select {
	case r := <-ch:
		return r, nil
	case <-time.After(time.Second):
		return Result{}, fmt.Errorf("timeout")
	}
}
```

如果 `slowQuery` 超过 1 秒才返回，外层已经返回，后台 goroutine 之后发送 `ch <- result` 时会永久阻塞。

更好的方向是让慢调用本身支持 context：

```go
func queryWithTimeout(parent context.Context, query string) (Result, error) {
	ctx, cancel := context.WithTimeout(parent, time.Second)
	defer cancel()

	return slowQueryContext(ctx, query)
}
```

如果必须用 goroutine 包装无法取消的调用，至少要使用带缓冲 channel 避免发送方卡死，但这只能避免发送泄漏，不能中断慢调用本身：

```go
ch := make(chan Result, 1)
go func() {
	ch <- slowQuery(query)
}()
```

真正的根因仍然是慢调用不可取消。

## 6. 如何写测试观察 goroutine 是否泄漏？

可以写一个趋势测试：记录基线，反复执行目标逻辑，等待清理，再检查 goroutine 数量是否持续上升。

```go
func TestNoGoroutineLeak(t *testing.T) {
	before := runtime.NumGoroutine()

	for i := 0; i < 100; i++ {
		ctx, cancel := context.WithCancel(context.Background())
		startReporter(ctx)
		cancel()
	}

	time.Sleep(100 * time.Millisecond)
	after := runtime.NumGoroutine()

	if after > before+5 {
		t.Fatalf("possible goroutine leak: before=%d after=%d", before, after)
	}
}
```

这个测试不是严格证明，因为运行时和测试框架也可能创建 goroutine。它适合做早期预警。

更严谨的排查方式是结合 goroutine profile，看新增 goroutine 的堆栈是否集中在你的代码里：

```sh
curl http://localhost:6060/debug/pprof/goroutine?debug=2
```

面试回答要强调：数量只是线索，堆栈才是定位依据。
