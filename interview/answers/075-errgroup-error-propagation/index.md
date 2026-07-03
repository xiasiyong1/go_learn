# 075. 并发错误

## 问题

多个 goroutine 并发执行时，错误收集和取消传播应该怎么做？

## 先给结论

`sync.WaitGroup` 只负责等待，不负责返回错误，也不负责取消其他 goroutine。只要并发任务之间存在“一个失败，其他任务就没必要继续”的关系，就要同时设计三件事：错误如何返回、其他 goroutine 如何取消、结果如何安全收集。工程里常用 `errgroup.WithContext` 解决这类问题。

## 1. WaitGroup 只能等，不能收错误

下面这段只能等待所有 goroutine 结束，无法知道哪个失败了。

```go
var wg sync.WaitGroup

for _, url := range urls {
	wg.Add(1)
	go func(url string) {
		defer wg.Done()
		_ = fetch(url) // 错误被丢掉了
	}(url)
}

wg.Wait()
```

如果任务失败会影响最终结果，不能只用 WaitGroup。

## 2. 用 channel 收错误要避免阻塞

一种手写方式是错误 channel。

```go
errCh := make(chan error, len(urls))
var wg sync.WaitGroup

for _, url := range urls {
	wg.Add(1)
	go func(url string) {
		defer wg.Done()
		errCh <- fetch(url)
	}(url)
}

wg.Wait()
close(errCh)

for err := range errCh {
	if err != nil {
		return err
	}
}
```

这里 `errCh` 用了缓冲，容量等于任务数。否则如果主 goroutine 还没接收，worker 发送错误时可能阻塞。

但这个写法仍有问题：一个任务失败后，其他任务还会继续跑。

## 3. 一个任务失败后要取消其他任务

用 context 传播取消。

```go
ctx, cancel := context.WithCancel(parent)
defer cancel()

errCh := make(chan error, len(urls))
var wg sync.WaitGroup

for _, url := range urls {
	wg.Add(1)
	go func(url string) {
		defer wg.Done()
		if err := fetchWithContext(ctx, url); err != nil {
			cancel()
			errCh <- err
			return
		}
		errCh <- nil
	}(url)
}

wg.Wait()
close(errCh)
```

注意：`cancel()` 只是发出信号。`fetchWithContext` 内部必须监听 context，否则取消不会生效。

## 4. errgroup 把等待、错误和取消合起来

`golang.org/x/sync/errgroup` 是常见工程写法。

```go
g, ctx := errgroup.WithContext(parent)

for _, url := range urls {
	url := url
	g.Go(func() error {
		return fetchWithContext(ctx, url)
	})
}

if err := g.Wait(); err != nil {
	return err
}
```

它做了几件事：

- 等待所有 `Go` 启动的函数返回。
- 返回第一个非 nil 错误。
- 使用 `WithContext` 时，任一函数返回错误会取消 context。

但它不会强制杀掉 goroutine。任务函数仍然要主动尊重 `ctx.Done()`。

## 5. 收集结果要避免数据竞争

错误写法：多个 goroutine 同时 append。

```go
var results []Result

g, ctx := errgroup.WithContext(parent)
for _, id := range ids {
	id := id
	g.Go(func() error {
		r, err := load(ctx, id)
		if err != nil {
			return err
		}
		results = append(results, r) // 数据竞争
		return nil
	})
}
```

如果结果顺序和输入一致，可以预分配并按下标写入。

```go
results := make([]Result, len(ids))

g, ctx := errgroup.WithContext(parent)
for i, id := range ids {
	i, id := i, id
	g.Go(func() error {
		r, err := load(ctx, id)
		if err != nil {
			return err
		}
		results[i] = r // 每个 goroutine 写不同下标
		return nil
	})
}

if err := g.Wait(); err != nil {
	return nil, err
}
return results, nil
```

如果多个 goroutine 可能写同一个 map 或 append 同一个 slice，就要加锁或改成 channel 汇总。

## 6. 任务很多时要限制并发

不要对成千上万个任务直接启动成千上万个 goroutine 去打下游。

```go
g, ctx := errgroup.WithContext(parent)
g.SetLimit(10)

for _, id := range ids {
	id := id
	g.Go(func() error {
		return process(ctx, id)
	})
}

return g.Wait()
```

如果不用 `errgroup.SetLimit`，也可以用信号量或 worker pool。

## 7. 面试时怎么答

可以这样回答：

- WaitGroup 只能等待，不能收错误，也不能取消。
- 手写错误 channel 要注意缓冲和关闭，否则容易阻塞或泄漏。
- 一个任务失败后，如果其他任务没有意义，应通过 context 取消。
- errgroup 能统一等待、返回第一个错误、取消 context。
- 任务函数必须主动监听 context，取消不是强杀。
- 结果收集要避免多个 goroutine 同时 append 或写 map。
- 任务很多时还要加并发上限。

## 面试追问

追问参考答案：[follow-ups.md](follow-ups.md)

- 为什么 WaitGroup 不适合处理错误传播？
- 错误 channel 为什么容易写出 goroutine 泄漏？
- errgroup 取消 context 后 goroutine 会立刻停止吗？
- 多 goroutine 收集结果有哪些安全写法？
- 只返回第一个错误够不够？
