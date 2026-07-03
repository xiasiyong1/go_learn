# 075. 并发错误 - 面试追问

## 1. 为什么 WaitGroup 不适合处理错误传播？

因为 `WaitGroup` 没有错误通道，也没有取消语义。

```go
var wg sync.WaitGroup

for _, task := range tasks {
	wg.Add(1)
	go func(task Task) {
		defer wg.Done()
		if err := task.Run(); err != nil {
			// 这里的 err 没有地方安全返回给调用方
		}
	}(task)
}

wg.Wait()
```

如果你只是“等所有任务结束”，WaitGroup 足够。如果你要“任一失败就返回错误，并通知其他任务停止”，就需要额外机制。

## 2. 错误 channel 为什么容易写出 goroutine 泄漏？

无缓冲错误 channel 如果没有接收者，发送方会阻塞。

```go
errCh := make(chan error)

go func() {
	errCh <- errors.New("failed") // 如果没人接收，会一直阻塞
}()
```

常见修正是使用足够缓冲，或者专门起 goroutine 等待关闭。

```go
errCh := make(chan error, len(tasks))

for _, task := range tasks {
	go func(task Task) {
		errCh <- task.Run()
	}(task)
}
```

但即使错误 channel 不阻塞，一个任务失败后其他任务仍会继续执行。要停止其他任务，需要 context。

## 3. errgroup 取消 context 后 goroutine 会立刻停止吗？

不会。context 只是一个取消信号，goroutine 要主动检查它。

不会及时退出的任务：

```go
g, ctx := errgroup.WithContext(parent)

g.Go(func() error {
	time.Sleep(10 * time.Second) // 不理 ctx
	return nil
})

_ = ctx
```

更好的写法：

```go
g.Go(func() error {
	select {
	case <-time.After(10 * time.Second):
		return nil
	case <-ctx.Done():
		return ctx.Err()
	}
})
```

真实 I/O 调用也要传 context，例如 `http.NewRequestWithContext`、`QueryContext`。

## 4. 多 goroutine 收集结果有哪些安全写法？

第一种：预分配 slice，每个 goroutine 写独立下标。

```go
results := make([]Result, len(ids))

for i, id := range ids {
	i, id := i, id
	g.Go(func() error {
		r, err := load(ctx, id)
		if err != nil {
			return err
		}
		results[i] = r
		return nil
	})
}
```

第二种：用 mutex 保护 append。

```go
var (
	mu      sync.Mutex
	results []Result
)

g.Go(func() error {
	r, err := load(ctx, id)
	if err != nil {
		return err
	}
	mu.Lock()
	results = append(results, r)
	mu.Unlock()
	return nil
})
```

第三种：用 channel 汇总，由单个 goroutine append。

```go
resultCh := make(chan Result)
```

选择标准：需要保序时用下标写入；不保序且数量不大时 mutex 简单；流水线式处理用 channel 更自然。

## 5. 只返回第一个错误够不够？

不一定。`errgroup` 默认返回第一个错误，适合“任一失败整体失败”的任务。

```go
if err := g.Wait(); err != nil {
	return err
}
```

如果业务需要展示所有失败项，就要聚合错误。

```go
var (
	mu   sync.Mutex
	errs []error
)

for _, task := range tasks {
	task := task
	wg.Add(1)
	go func() {
		defer wg.Done()
		if err := task.Run(); err != nil {
			mu.Lock()
			errs = append(errs, err)
			mu.Unlock()
		}
	}()
}
wg.Wait()

return errors.Join(errs...)
```

面试回答要说清楚业务语义：是 fail-fast，还是尽量执行完并汇总所有错误。
