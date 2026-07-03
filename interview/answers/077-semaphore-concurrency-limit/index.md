# 077. 并发限制

## 问题

Go 中如何用信号量限制并发？和 worker pool 有什么区别？

## 先给结论

信号量限制的是“同时进入某段代码的数量”，worker pool 限制的是“同时执行任务的 worker 数量”。简单信号量可以用带缓冲 channel 实现：发送表示获取许可，接收表示释放许可。关键是 acquire/release 必须成对，并且等待许可时最好支持 context 取消。

## 1. 用带缓冲 channel 实现信号量

最多允许 3 个任务同时执行：

```go
sem := make(chan struct{}, 3)

for _, job := range jobs {
	job := job
	sem <- struct{}{} // acquire
	go func() {
		defer func() { <-sem }() // release
		process(job)
	}()
}
```

当 channel 满了，新的发送会阻塞，于是并发数被限制住。

## 2. acquire/release 必须成对

错误路径忘记释放，会导致所有后续任务卡死。

```go
sem <- struct{}{}

if err := do(); err != nil {
	return err // 忘记 <-sem
}

<-sem
```

更稳的写法：

```go
sem <- struct{}{}
defer func() { <-sem }()

if err := do(); err != nil {
	return err
}
```

如果 goroutine 内部可能 panic，`defer` 仍然能释放许可。

```go
go func() {
	sem <- struct{}{}
	defer func() { <-sem }()
	mayPanic()
}()
```

注意：这只释放许可，不代表 panic 被恢复。如果需要服务不崩，还要 recover。

## 3. 获取许可时支持 context

如果请求已经取消，还一直等许可就没有意义。

```go
func acquire(ctx context.Context, sem chan struct{}) error {
	select {
	case sem <- struct{}{}:
		return nil
	case <-ctx.Done():
		return ctx.Err()
	}
}

func release(sem chan struct{}) {
	<-sem
}
```

使用：

```go
if err := acquire(ctx, sem); err != nil {
	return err
}
defer release(sem)

return callDownstream(ctx)
```

这能防止大量已取消请求排队等待许可。

## 4. 信号量和 worker pool 的区别

信号量更适合包住一段受限逻辑。

```go
if err := acquire(ctx, sem); err != nil {
	return err
}
defer release(sem)

return db.QueryRowContext(ctx, query).Scan(&v)
```

worker pool 更适合处理明确的任务队列和 worker 生命周期。

```go
jobs := make(chan Job)

for i := 0; i < workerN; i++ {
	go func() {
		for job := range jobs {
			process(job)
		}
	}()
}
```

如果调用方仍然是同步请求，只想限制某个下游并发，信号量通常更轻。如果任务需要排队、关闭、结果汇总，worker pool 更清楚。

## 5. 并发限制不是速率限制

并发限制控制“同时有多少个请求在执行”。

```go
sem := make(chan struct{}, 10)
```

速率限制控制“单位时间允许多少请求”。

```go
ticker := time.NewTicker(100 * time.Millisecond)
defer ticker.Stop()

for _, job := range jobs {
	<-ticker.C // 每 100ms 放一个
	go process(job)
}
```

两者可以同时存在：例如最多 10 个并发，同时每秒不超过 100 个请求。

## 6. 加权信号量适合不同任务消耗不同资源

带缓冲 channel 每个任务通常占 1 个许可。如果任务成本不同，可以使用加权信号量思想，例如大任务占更多许可。标准库没有加权信号量，常见实现来自 `golang.org/x/sync/semaphore`。

```go
sem := semaphore.NewWeighted(10)

if err := sem.Acquire(ctx, 3); err != nil {
	return err
}
defer sem.Release(3)
```

面试中不要求背库名，但要知道普通 channel 信号量表达的是“每个任务成本相同”的简化模型。

## 7. 面试时怎么答

可以这样回答：

- 信号量限制同时进入某段代码的 goroutine 数。
- 带缓冲 channel 可以实现简单信号量：发送获取许可，接收释放许可。
- acquire/release 必须成对，通常用 defer 保证释放。
- 获取许可时应支持 context，避免取消请求继续排队。
- 信号量限制并发，不等于速率限制。
- worker pool 更适合任务队列，信号量更适合保护某段资源访问。

## 面试追问

追问参考答案：[follow-ups.md](follow-ups.md)

- 为什么带缓冲 channel 可以当信号量？
- 忘记释放许可会发生什么？
- 如何让等待许可支持 context 取消？
- 信号量和 worker pool 怎么选？
- 并发限制和限流有什么区别？
