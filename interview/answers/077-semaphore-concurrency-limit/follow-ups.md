# 077. 并发限制 - 面试追问

## 1. 为什么带缓冲 channel 可以当信号量？

因为 channel 容量可以表示可用许可数量。写入一个空结构体表示占用一个许可，读出一个值表示释放一个许可。

```go
sem := make(chan struct{}, 2)

sem <- struct{}{} // 第 1 个许可
sem <- struct{}{} // 第 2 个许可

// sem <- struct{}{} // 第 3 个会阻塞

<-sem // 释放 1 个许可
```

`struct{}{}` 不携带数据，通常只用来表达信号。

## 2. 忘记释放许可会发生什么？

容量会被永久占用，最终所有 goroutine 都阻塞在 acquire。

```go
func bad(sem chan struct{}) error {
	sem <- struct{}{}

	if err := do(); err != nil {
		return err // 泄漏许可
	}

	<-sem
	return nil
}
```

正确写法：

```go
func good(sem chan struct{}) error {
	sem <- struct{}{}
	defer func() { <-sem }()

	return do()
}
```

如果 do 里 panic，defer 仍会执行。但 panic 是否恢复，要看你是否额外 recover。

## 3. 如何让等待许可支持 context 取消？

用 select 同时等待许可和 context。

```go
func acquire(ctx context.Context, sem chan struct{}) error {
	select {
	case sem <- struct{}{}:
		return nil
	case <-ctx.Done():
		return ctx.Err()
	}
}
```

使用：

```go
if err := acquire(ctx, sem); err != nil {
	return err
}
defer func() { <-sem }()

return call(ctx)
```

这对 HTTP 服务很重要：客户端已经断开或请求超时，就不应该继续排队等待下游许可。

## 4. 信号量和 worker pool 怎么选？

同步请求里限制某段下游调用，用信号量。

```go
func (s *Service) Get(ctx context.Context, id string) error {
	if err := acquire(ctx, s.dbSem); err != nil {
		return err
	}
	defer release(s.dbSem)

	return s.repo.Load(ctx, id)
}
```

批量后台任务、持续消费队列，用 worker pool。

```go
jobs := make(chan Job)

for i := 0; i < 10; i++ {
	go func() {
		for job := range jobs {
			process(job)
		}
	}()
}
```

判断标准：你是在保护一段代码的同时执行数，还是在设计一套任务队列和 worker 生命周期。

## 5. 并发限制和限流有什么区别？

并发限制看的是同时执行数量。

```go
sem := make(chan struct{}, 10) // 最多 10 个同时执行
```

限流看的是单位时间允许多少请求。

```go
limiter := time.NewTicker(10 * time.Millisecond)
defer limiter.Stop()

for _, req := range requests {
	<-limiter.C
	go handle(req)
}
```

举例：一个请求耗时 1 秒，并发限制 10，理论吞吐约 10 QPS；如果请求耗时变成 100ms，同样并发限制 10，理论吞吐会变成 100 QPS。限流则直接控制速率。

稳定性设计里，两者经常一起用。
