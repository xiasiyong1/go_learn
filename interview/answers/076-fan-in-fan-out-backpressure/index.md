# 076. 并发模式

## 问题

fan-in、fan-out 和背压在 Go 并发里应该怎么设计？

## 先给结论

fan-out 是把任务分发给多个 worker 并行处理，fan-in 是把多个来源的结果汇聚到一个出口。真正需要讲深的是背压：当生产快、消费慢、下游失败或消费者提前退出时，代码不能无限堆积，也不能把 goroutine 卡死在发送或接收上。

## 1. fan-out：多个 worker 消费同一个任务源

```go
func worker(ctx context.Context, jobs <-chan Job, results chan<- Result) error {
	for {
		select {
		case <-ctx.Done():
			return ctx.Err()
		case job, ok := <-jobs:
			if !ok {
				return nil
			}
			result := handle(job)
			select {
			case results <- result:
			case <-ctx.Done():
				return ctx.Err()
			}
		}
	}
}
```

关键点：

- worker 从同一个 `jobs` channel 取任务。
- `jobs` 应该由生产者关闭。
- 发送结果时也要监听 `ctx.Done()`，否则消费者退出后 worker 可能卡住。

## 2. fan-in：多个输入合并成一个输出

合并多个 channel 时，不能提前关闭输出 channel。必须等所有转发 goroutine 结束。

```go
func merge[T any](ctx context.Context, inputs ...<-chan T) <-chan T {
	out := make(chan T)
	var wg sync.WaitGroup

	for _, in := range inputs {
		in := in
		wg.Add(1)
		go func() {
			defer wg.Done()
			for v := range in {
				select {
				case out <- v:
				case <-ctx.Done():
					return
				}
			}
		}()
	}

	go func() {
		wg.Wait()
		close(out)
	}()

	return out
}
```

这里关闭 `out` 的只能是协调者，不能让任意一个输入结束时就关闭 `out`。

## 3. 背压是什么

背压就是让下游处理能力反向影响上游生产速度。无缓冲或小缓冲 channel 天然会形成背压。

```go
jobs := make(chan Job) // 无缓冲

go producer(jobs)
go consumer(jobs)
```

如果 consumer 慢，producer 发送会阻塞。

有缓冲 channel 可以吸收突发：

```go
jobs := make(chan Job, 100)
```

但缓冲不是吞吐能力。消费者长期慢，缓冲迟早会满。缓冲过大还会增加内存和尾延迟。

## 4. 消费者提前退出时必须取消上游

错误示例：只取第一个结果后返回。

```go
func first(results <-chan Result) Result {
	return <-results
}
```

如果上游还在继续发送，可能永远阻塞。

更好的写法是带 context：

```go
func first(ctx context.Context, cancel context.CancelFunc, results <-chan Result) (Result, error) {
	defer cancel()

	select {
	case r := <-results:
		return r, nil
	case <-ctx.Done():
		return Result{}, ctx.Err()
	}
}
```

所有上游发送点都要监听 `ctx.Done()`，取消才有意义。

## 5. worker 数不是越多越好

CPU 密集任务通常接近 `GOMAXPROCS`。I/O 密集任务可能更多，但还要受下游连接池、QPS、数据库连接数限制。

```go
workerN := 10
for i := 0; i < workerN; i++ {
	go worker(ctx, jobs, results)
}
```

如果下游数据库只有 20 个连接，开 1000 个 worker 只会制造排队、超时和更高内存。

## 6. 完整的关闭顺序

典型顺序：

```go
jobs := make(chan Job)
results := make(chan Result)

// 生产者负责关闭 jobs
go func() {
	defer close(jobs)
	for _, job := range allJobs {
		select {
		case jobs <- job:
		case <-ctx.Done():
			return
		}
	}
}()

// worker 结束后，由协调者关闭 results
var wg sync.WaitGroup
for i := 0; i < workerN; i++ {
	wg.Add(1)
	go func() {
		defer wg.Done()
		worker(ctx, jobs, results)
	}()
}

go func() {
	wg.Wait()
	close(results)
}()
```

原则：谁发送，谁关闭；多个发送者时，由协调者等所有发送者结束后关闭。

## 7. 面试时怎么答

可以这样回答：

- fan-out 是多个 worker 从同一个任务源消费。
- fan-in 是把多个输入合并到一个输出，输出 channel 只能在所有发送者结束后关闭。
- 背压是下游慢时让上游阻塞或降速，channel 缓冲只能吸收突发，不能解决长期处理能力不足。
- 所有发送和接收点都要考虑 context 取消，防止消费者提前退出导致 goroutine 泄漏。
- worker 数要根据 CPU、I/O、下游容量和延迟目标决定。

## 面试追问

追问参考答案：[follow-ups.md](follow-ups.md)

- fan-out 和 worker pool 是一回事吗？
- fan-in 的输出 channel 应该由谁关闭？
- 消费者只要第一个结果时，如何避免上游泄漏？
- channel 缓冲区越大越好吗？
- 如何判断 worker 数量设置得是否合理？
