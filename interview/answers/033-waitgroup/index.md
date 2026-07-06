# 033. WaitGroup

## 问题

`WaitGroup` 怎么用？常见错误有哪些？

## 先给结论

`sync.WaitGroup` 用来等待一组 goroutine 完成。它只负责“计数等待”，不负责：

- 收集错误。
- 传播取消。
- 限制并发数。
- 传递返回值。

基本规则：

- 启动 goroutine 前调用 `Add(1)`。
- goroutine 内部 `defer Done()`。
- `Done()` 等价于 `Add(-1)`。
- 计数变成负数会 panic。
- 不要复制使用中的 WaitGroup。

## 正确基本写法

```go
var wg sync.WaitGroup

for _, job := range jobs {
	wg.Add(1)
	go func(job Job) {
		defer wg.Done()
		process(job)
	}(job)
}

wg.Wait()
```

`Add(1)` 放在 goroutine 启动前，确保 `Wait()` 看到准确的任务数量。

## 不要在 goroutine 里 Add

错误示例：

```go
var wg sync.WaitGroup

for _, job := range jobs {
	go func(job Job) {
		wg.Add(1) // 错误
		defer wg.Done()
		process(job)
	}(job)
}

wg.Wait()
```

`Wait()` 可能在某些 goroutine 执行 `Add(1)` 前就看到计数为 0 并返回，导致任务还没结束函数就继续往下走。

正确写法仍然是先 Add，再 go。

## Done 要覆盖所有退出路径

```go
go func() {
	defer wg.Done()

	if err := step1(); err != nil {
		return
	}
	step2()
}()
```

用 `defer wg.Done()` 可以避免多个 return 分支漏掉 Done。

如果漏掉：

```go
go func() {
	if err := step1(); err != nil {
		return // 忘记 Done，Wait 永久阻塞
	}
	wg.Done()
}()
```

`wg.Wait()` 会一直等。

## Done 调多会 panic

```go
var wg sync.WaitGroup
wg.Add(1)
wg.Done()
wg.Done() // panic: negative WaitGroup counter
```

这个 panic 通常说明 Add/Done 配对关系不清楚。

## WaitGroup 不处理错误

错误示例：

```go
var wg sync.WaitGroup

for _, job := range jobs {
	wg.Add(1)
	go func(job Job) {
		defer wg.Done()
		_ = process(job) // 错误被吞掉
	}(job)
}

wg.Wait()
```

简单错误收集可以用 channel：

```go
errCh := make(chan error, len(jobs))
var wg sync.WaitGroup

for _, job := range jobs {
	wg.Add(1)
	go func(job Job) {
		defer wg.Done()
		errCh <- process(job)
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

如果希望第一个错误取消其他任务，更适合 `errgroup.WithContext`。

## WaitGroup 不限制并发

下面会一次性启动很多 goroutine：

```go
for _, job := range jobs {
	wg.Add(1)
	go func(job Job) {
		defer wg.Done()
		process(job)
	}(job)
}
```

如果任务很多，要配合信号量或 worker pool：

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

WaitGroup 只等任务结束，不控制同时运行多少任务。

## 不要复制 WaitGroup

```go
func Bad(wg sync.WaitGroup) {
	wg.Done()
}
```

这会复制 WaitGroup，修改的是副本。

应该传指针：

```go
func Good(wg *sync.WaitGroup) {
	wg.Done()
}
```

更常见的是把 Add/Done/Wait 控制在同一个函数里，减少跨函数传 WaitGroup。

## 面试追问

追问参考答案：[follow-ups.md](follow-ups.md)

- `Add` 为什么应该在启动 goroutine 前调用？
- `Done` 和 `Add(-1)` 是什么关系？
- WaitGroup 能不能收集 goroutine 的错误？
- WaitGroup 能不能限制并发数？
- WaitGroup 可以复用吗？
- 为什么不能复制使用中的 WaitGroup？
