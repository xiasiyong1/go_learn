# 036. worker pool 和 pipeline

## 问题

worker pool 和 pipeline 怎么设计？

## 先给结论

worker pool 用固定数量 worker 处理任务，核心目的是限制并发度和资源占用。

pipeline 把处理过程拆成多个阶段，每个阶段从上游接收数据，处理后发送到下游。

面试重点不是画出 channel，而是讲清楚：

- 谁关闭 channel。
- 错误如何返回。
- 取消如何传播。
- 下游提前退出时上游如何停止。
- worker 数量和缓冲大小如何决定。

## 简单 worker pool

```go
type Job struct {
	ID int
}

type Result struct {
	ID  int
	Err error
}

func RunPool(ctx context.Context, jobs []Job, workerN int) []Result {
	jobCh := make(chan Job)
	resultCh := make(chan Result)

	var wg sync.WaitGroup
	for i := 0; i < workerN; i++ {
		wg.Add(1)
		go func() {
			defer wg.Done()
			for job := range jobCh {
				resultCh <- handle(ctx, job)
			}
		}()
	}

	go func() {
		defer close(jobCh)
		for _, job := range jobs {
			jobCh <- job
		}
	}()

	go func() {
		wg.Wait()
		close(resultCh)
	}()

	var results []Result
	for r := range resultCh {
		results = append(results, r)
	}
	return results
}
```

这个版本能展示结构，但还不完整：发送任务和发送结果都没有监听 `ctx.Done()`，如果下游提前返回，可能泄漏。面试中要能指出这一点。

## 支持取消的发送

发送任务时监听 context：

```go
go func() {
	defer close(jobCh)
	for _, job := range jobs {
		select {
		case jobCh <- job:
		case <-ctx.Done():
			return
		}
	}
}()
```

worker 发送结果时也监听：

```go
select {
case resultCh <- handle(ctx, job):
case <-ctx.Done():
	return
}
```

每个可能阻塞的 send/receive 都要考虑取消。

## worker 数量怎么定

CPU 密集型任务：

```go
workerN := runtime.GOMAXPROCS(0)
```

I/O 密集型任务要看外部依赖：

- 数据库连接池上限。
- RPC 对端限流。
- 磁盘或网络吞吐。
- 单个任务平均耗时。

不要随手写 `1000`。worker 数本质上是资源上限。

## pipeline 示例

```go
func gen(ctx context.Context, nums ...int) <-chan int {
	out := make(chan int)
	go func() {
		defer close(out)
		for _, n := range nums {
			select {
			case out <- n:
			case <-ctx.Done():
				return
			}
		}
	}()
	return out
}

func square(ctx context.Context, in <-chan int) <-chan int {
	out := make(chan int)
	go func() {
		defer close(out)
		for n := range in {
			select {
			case out <- n * n:
			case <-ctx.Done():
				return
			}
		}
	}()
	return out
}
```

使用：

```go
ctx, cancel := context.WithCancel(context.Background())
defer cancel()

for n := range square(ctx, gen(ctx, 1, 2, 3)) {
	fmt.Println(n)
}
```

每个阶段负责关闭自己的输出 channel。每个阶段都监听取消。

## 下游提前退出要取消上游

错误示例：

```go
for n := range square(ctx, gen(ctx, nums...)) {
	if n > 100 {
		return n // 上游可能还在发送
	}
}
```

正确方向：

```go
ctx, cancel := context.WithCancel(parent)
defer cancel()

for n := range square(ctx, gen(ctx, nums...)) {
	if n > 100 {
		cancel()
		return n
	}
}
```

下游不再接收时，要通知上游停止发送。

## 错误传播

简单 worker pool 可以用 error channel 或 result 包含 error。更复杂场景可以用 `errgroup.WithContext`。

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

`errgroup` 会收集第一个错误，并配合 context 取消其他任务。

## 面试追问

追问参考答案：[follow-ups.md](follow-ups.md)

- worker pool 解决什么问题？
- worker 数量应该怎么定？
- pipeline 每个阶段应该由谁关闭 channel？
- 下游提前退出时为什么会上游泄漏？
- worker pool 如何处理错误和取消？
- 有缓冲 channel 在 worker pool 里解决什么，又掩盖什么？
