# 033. WaitGroup - 面试追问

## 1. `Add` 为什么应该在启动 goroutine 前调用？

因为 `Wait()` 可能在 goroutine 运行 `Add` 前就看到计数为 0 并返回。

错误示例：

```go
var wg sync.WaitGroup

go func() {
	wg.Add(1)
	defer wg.Done()
	doWork()
}()

wg.Wait()
```

正确写法：

```go
var wg sync.WaitGroup

wg.Add(1)
go func() {
	defer wg.Done()
	doWork()
}()

wg.Wait()
```

原则是：任务数量由启动方确定，启动方先 Add。

## 2. `Done` 和 `Add(-1)` 是什么关系？

`Done()` 等价于 `Add(-1)`。

```go
wg.Add(1)
wg.Done()
```

等价于：

```go
wg.Add(1)
wg.Add(-1)
```

如果计数变成负数，会 panic：

```go
var wg sync.WaitGroup
wg.Done() // panic
```

所以 Add 和 Done 必须配对。goroutine 内部通常写 `defer wg.Done()`。

## 3. WaitGroup 能不能收集 goroutine 的错误？

不能。它只等待，不传值。

错误会被吞：

```go
go func() {
	defer wg.Done()
	_ = doWork()
}()
```

简单方式用 error channel：

```go
errCh := make(chan error, len(jobs))

for _, job := range jobs {
	wg.Add(1)
	go func(job Job) {
		defer wg.Done()
		errCh <- doWork(job)
	}(job)
}

wg.Wait()
close(errCh)

for err := range errCh {
	if err != nil {
		return err
	}
}
```

如果还要“第一个错误取消其他任务”，用 `errgroup.WithContext` 更合适。

## 4. WaitGroup 能不能限制并发数？

不能。它只等待已经启动的任务。

下面会启动 `len(jobs)` 个 goroutine：

```go
for _, job := range jobs {
	wg.Add(1)
	go func(job Job) {
		defer wg.Done()
		process(job)
	}(job)
}
```

限制并发可以加信号量：

```go
sem := make(chan struct{}, 10)

for _, job := range jobs {
	wg.Add(1)
	go func(job Job) {
		defer wg.Done()

		sem <- struct{}{}
		defer func() { <-sem }()

		process(job)
	}(job)
}
```

或者使用固定数量 worker。

## 5. WaitGroup 可以复用吗？

可以，但必须等上一轮 `Wait()` 返回后，再开始下一轮 Add。

```go
var wg sync.WaitGroup

runBatch := func(jobs []Job) {
	for _, job := range jobs {
		wg.Add(1)
		go func(job Job) {
			defer wg.Done()
			process(job)
		}(job)
	}
	wg.Wait()
}

runBatch(batch1)
runBatch(batch2)
```

不要在一轮 Wait 尚未完成时混入下一轮 Add，这会让任务边界混乱。

## 6. 为什么不能复制使用中的 WaitGroup？

WaitGroup 内部有状态。复制后，Add/Done/Wait 操作可能落在不同副本上。

错误示例：

```go
func Done(wg sync.WaitGroup) {
	wg.Done()
}
```

正确：

```go
func Done(wg *sync.WaitGroup) {
	wg.Done()
}
```

更好的设计是不要把 WaitGroup 到处传，让创建 goroutine 的函数自己负责 Add、Done 和 Wait。

可以用工具辅助发现复制锁类问题：

```sh
go vet -copylocks ./...
```
