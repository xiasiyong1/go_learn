# 036. worker pool 和 pipeline - 面试追问

## 1. worker pool 解决什么问题？

解决并发上限问题。

不限制并发：

```go
for _, job := range jobs {
	go process(job)
}
```

任务很多时，可能打爆数据库、RPC、内存或调度器。

worker pool：

```go
jobsCh := make(chan Job)
for i := 0; i < workerN; i++ {
	go func() {
		for job := range jobsCh {
			process(job)
		}
	}()
}
```

同时处理的任务最多是 `workerN`。

## 2. worker 数量应该怎么定？

CPU 密集型接近 CPU 并行度：

```go
workerN := runtime.GOMAXPROCS(0)
```

I/O 密集型要看外部系统：

- 数据库连接池大小。
- 下游 API 限流。
- 单任务延迟。
- 允许的排队时间。

不要把 workerN 当纯性能参数。它也是保护下游的资源上限。

## 3. pipeline 每个阶段应该由谁关闭 channel？

每个阶段关闭自己的输出 channel。

```go
func stage(ctx context.Context, in <-chan int) <-chan int {
	out := make(chan int)
	go func() {
		defer close(out)
		for v := range in {
			select {
			case out <- transform(v):
			case <-ctx.Done():
				return
			}
		}
	}()
	return out
}
```

下游不应该关闭上游的输出 channel，因为上游可能还在发送。

## 4. 下游提前退出时为什么会上游泄漏？

因为上游可能阻塞在发送。

```go
func gen() <-chan int {
	out := make(chan int)
	go func() {
		for i := 0; ; i++ {
			out <- i
		}
	}()
	return out
}

func first() int {
	return <-gen()
}
```

`first` 只读一个值就返回，`gen` 还会继续发送下一个值，然后阻塞。

修正：传 context 并取消。

```go
ctx, cancel := context.WithCancel(parent)
defer cancel()

out := gen(ctx)
v := <-out
cancel()
return v
```

## 5. worker pool 如何处理错误和取消？

简单方式：结果里带 error。

```go
type Result struct {
	Value Value
	Err   error
}
```

如果第一个错误应该取消其他任务，用 `errgroup.WithContext`：

```go
g, ctx := errgroup.WithContext(parent)

for _, job := range jobs {
	job := job
	g.Go(func() error {
		return process(ctx, job)
	})
}

if err := g.Wait(); err != nil {
	return err
}
```

关键是任务内部也要监听 ctx，否则取消不会生效。

## 6. 有缓冲 channel 在 worker pool 里解决什么，又掩盖什么？

它能吸收短时间突发，让生产者和 worker 解耦一点。

```go
jobs := make(chan Job, 100)
```

但它不能提升长期处理能力。如果 worker 太慢，缓冲迟早会满。

缓冲过大还会掩盖问题：

- 排队时间变长。
- 内存占用上升。
- 上游更晚感知下游变慢。

所以容量应该来自业务允许排队的数量和时间，而不是越大越好。
