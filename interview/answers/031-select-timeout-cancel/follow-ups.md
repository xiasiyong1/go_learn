# 031. select、超时和取消 - 面试追问

## 1. `select` 在多个 case 同时 ready 时如何选择？

会伪随机选择一个 ready case，不能依赖源码顺序。

```go
ch1 := make(chan int, 1)
ch2 := make(chan int, 1)
ch1 <- 1
ch2 <- 2

select {
case <-ch1:
	fmt.Println("ch1")
case <-ch2:
	fmt.Println("ch2")
}
```

如果业务需要优先级，要显式写逻辑，而不是依赖 case 顺序。

```go
select {
case v := <-high:
	handle(v)
default:
	select {
	case v := <-high:
		handle(v)
	case v := <-low:
		handle(v)
	}
}
```

这种优先级代码要写测试，否则很容易引入饥饿或漏处理。

## 2. `default` 有什么作用，为什么可能导致忙轮询？

`default` 让 select 在没有 case ready 时立即返回。

```go
select {
case v := <-ch:
	return v
default:
	return zero
}
```

适合非阻塞尝试。

错误用法：

```go
for {
	select {
	case job := <-jobs:
		handle(job)
	default:
	}
}
```

没有 job 时循环不停空转，会吃 CPU。

如果想等待任务，不要写 default：

```go
for {
	select {
	case job := <-jobs:
		handle(job)
	case <-ctx.Done():
		return
	}
}
```

## 3. 单次超时和循环超时分别应该怎么写？

单次等待可以用 `time.After`：

```go
select {
case result := <-resultCh:
	return result, nil
case <-time.After(time.Second):
	return Result{}, fmt.Errorf("timeout")
}
```

循环周期任务用 ticker：

```go
ticker := time.NewTicker(time.Second)
defer ticker.Stop()

for {
	select {
	case job := <-jobs:
		handle(job)
	case <-ticker.C:
		report()
	case <-ctx.Done():
		return ctx.Err()
	}
}
```

请求级超时优先用 context，把同一个 deadline 传到下游：

```go
ctx, cancel := context.WithTimeout(parent, time.Second)
defer cancel()

return call(ctx)
```

## 4. 为什么循环里反复 `time.After` 要谨慎？

`time.After` 每次调用都会创建 timer。

```go
for {
	select {
	case <-time.After(time.Millisecond):
		tick()
	case <-ctx.Done():
		return
	}
}
```

高频循环里会产生额外分配和 timer 管理成本。

如果是周期任务，用 ticker：

```go
ticker := time.NewTicker(time.Millisecond)
defer ticker.Stop()

for {
	select {
	case <-ticker.C:
		tick()
	case <-ctx.Done():
		return
	}
}
```

如果是每次操作独立 deadline，可以创建并复用 `Timer`，但要正确处理 Stop 和 drain，代码复杂度更高。

## 5. nil channel 在 `select` 里有什么用途？

nil channel 永远不会 ready，所以可以用来动态禁用 case。

```go
for ch1 != nil || ch2 != nil {
	select {
	case v, ok := <-ch1:
		if !ok {
			ch1 = nil
			continue
		}
		handle(v)
	case v, ok := <-ch2:
		if !ok {
			ch2 = nil
			continue
		}
		handle(v)
	}
}
```

如果不把关闭的 channel 设为 nil，后续接收会一直立即返回零值和 `ok=false`，可能导致循环异常。

## 6. 如何保证取消信号不会被阻塞点漏掉？

每个可能阻塞的发送、接收、I/O 都要考虑 context。

发送可能阻塞：

```go
select {
case out <- result:
case <-ctx.Done():
	return ctx.Err()
}
```

接收可能阻塞：

```go
select {
case job := <-jobs:
	handle(job)
case <-ctx.Done():
	return ctx.Err()
}
```

外部调用也要传 context：

```go
req, err := http.NewRequestWithContext(ctx, http.MethodGet, url, nil)
```

只在最外层创建 context，但内部阻塞点不监听，取消不会真正生效。
