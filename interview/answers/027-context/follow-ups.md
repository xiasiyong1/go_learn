# 027. context - 面试追问

## 1. context 取消后 goroutine 会自动退出吗？

不会。context 只是取消信号，goroutine 必须主动监听。

错误示例：

```go
func Start(ctx context.Context) {
	go func() {
		for {
			doWork()
		}
	}()
}
```

外部调用 `cancel()` 后，这个 goroutine 仍然继续跑。

正确写法：

```go
func Start(ctx context.Context) {
	go func() {
		for {
			select {
			case <-ctx.Done():
				return
			default:
				doWork()
			}
		}
	}()
}
```

如果 `doWork` 自己会阻塞，也要让它接收 context：

```go
func Start(ctx context.Context) {
	go func() {
		for {
			if err := doWork(ctx); err != nil {
				return
			}
		}
	}()
}
```

## 2. 为什么创建 `WithTimeout` 后仍然要调用 `cancel()`？

因为 `WithTimeout` 内部会创建 timer 和子 context。调用 `cancel()` 可以及时释放相关资源，并通知子 context。

```go
ctx, cancel := context.WithTimeout(parent, time.Second)
defer cancel()

return call(ctx)
```

即使 `call` 很快返回，也应该 `defer cancel()`。

不推荐：

```go
ctx, _ := context.WithTimeout(parent, time.Second)
return call(ctx)
```

这会让 timer 直到超时或父 context 取消才释放。

## 3. `context.Canceled` 和 `context.DeadlineExceeded` 有什么区别？

`context.Canceled` 表示被主动取消。

```go
ctx, cancel := context.WithCancel(context.Background())
cancel()

fmt.Println(ctx.Err()) // context canceled
```

`context.DeadlineExceeded` 表示超过 deadline 或 timeout。

```go
ctx, cancel := context.WithTimeout(context.Background(), time.Nanosecond)
defer cancel()

<-ctx.Done()
fmt.Println(ctx.Err()) // context deadline exceeded
```

真实服务里两者含义不同：客户端断开、上游主动取消通常是 canceled；依赖慢、处理超时通常是 deadline exceeded。

## 4. 为什么不建议把 context 存进结构体？

因为 context 是请求范围的。不同请求应该有不同的超时、取消、trace 信息。

不推荐：

```go
type Client struct {
	ctx context.Context
}

func (c *Client) Get(url string) error {
	req, _ := http.NewRequestWithContext(c.ctx, http.MethodGet, url, nil)
	_, err := http.DefaultClient.Do(req)
	return err
}
```

推荐：

```go
type Client struct{}

func (c *Client) Get(ctx context.Context, url string) error {
	req, err := http.NewRequestWithContext(ctx, http.MethodGet, url, nil)
	if err != nil {
		return err
	}
	_, err = http.DefaultClient.Do(req)
	return err
}
```

结构体保存依赖，方法参数传 context。这个边界更清楚。

## 5. `context.Value` 适合放什么，不适合放什么？

适合放请求范围元数据，例如 trace id、request id、认证结果。

```go
type requestIDKey struct{}

func WithRequestID(ctx context.Context, id string) context.Context {
	return context.WithValue(ctx, requestIDKey{}, id)
}
```

不适合放业务必填参数：

```go
func Pay(ctx context.Context) error {
	orderID := ctx.Value("order_id").(int64) // 不推荐
	_ = orderID
	return nil
}
```

业务参数应该显式传：

```go
func Pay(ctx context.Context, orderID int64) error {
	return nil
}
```

这样调用方、测试和静态检查都更清楚。

## 6. 如何写一个支持取消的 worker？

要在所有阻塞点监听 `ctx.Done()`。

```go
func Worker(ctx context.Context, jobs <-chan Job, results chan<- Result) {
	for {
		select {
		case <-ctx.Done():
			return
		case job, ok := <-jobs:
			if !ok {
				return
			}

			result, err := handle(ctx, job)
			select {
			case results <- Result{Value: result, Err: err}:
			case <-ctx.Done():
				return
			}
		}
	}
}
```

这里接收 job 和发送 result 都可能阻塞，所以两个地方都要考虑取消。
