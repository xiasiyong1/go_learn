# 027. context

## 问题

`context.Context` 应该怎么用？常见误用有哪些？

## 先给结论

`context.Context` 用来在调用链上传递取消信号、超时、截止时间和请求范围值。

它不负责强制杀死 goroutine，也不是可选参数容器。代码必须主动监听 `ctx.Done()`，阻塞调用也必须接收 context，取消才真正生效。

常见规则：

- `ctx` 通常作为函数第一个参数。
- 不要把 `context.Context` 存进结构体长期持有。
- 派生出带 cancel/timeout 的 context 后，要调用 `cancel()`。
- `context.Value` 只放请求范围元数据，不放业务必填参数。

## 基本用法

```go
func FetchUser(ctx context.Context, id int64) (*User, error) {
	req, err := http.NewRequestWithContext(ctx, http.MethodGet, userURL(id), nil)
	if err != nil {
		return nil, err
	}

	resp, err := http.DefaultClient.Do(req)
	if err != nil {
		return nil, err
	}
	defer resp.Body.Close()

	return decodeUser(resp.Body)
}
```

调用方设置超时：

```go
ctx, cancel := context.WithTimeout(context.Background(), 2*time.Second)
defer cancel()

u, err := FetchUser(ctx, 1001)
```

`defer cancel()` 即使在正常返回时也要调用，因为它会释放 timer 和子 context 相关资源。

## `WithCancel`、`WithTimeout`、`WithDeadline`

手动取消：

```go
ctx, cancel := context.WithCancel(parent)
defer cancel()

go func() {
	if needStop() {
		cancel()
	}
}()
```

相对超时：

```go
ctx, cancel := context.WithTimeout(parent, 3*time.Second)
defer cancel()
```

绝对截止时间：

```go
deadline := time.Now().Add(3 * time.Second)
ctx, cancel := context.WithDeadline(parent, deadline)
defer cancel()
```

`WithTimeout` 本质上是设置“从现在开始多久后取消”；`WithDeadline` 是“到某个具体时间点取消”。

## context 是取消信号，不是强制中断

错误示例：

```go
func Worker(ctx context.Context) {
	for {
		doWork() // 永远不检查 ctx
	}
}
```

即使外部调用 `cancel()`，这个 goroutine 也不会自动退出。

正确方向：

```go
func Worker(ctx context.Context, jobs <-chan Job) {
	for {
		select {
		case <-ctx.Done():
			return
		case job, ok := <-jobs:
			if !ok {
				return
			}
			handle(job)
		}
	}
}
```

如果 `handle` 里也可能阻塞，要继续传入 context：

```go
func Worker(ctx context.Context, jobs <-chan Job) {
	for {
		select {
		case <-ctx.Done():
			return
		case job := <-jobs:
			_ = handle(ctx, job)
		}
	}
}
```

取消信号必须传到真正可能阻塞的位置。

## 用 `Err()` 区分取消原因

```go
select {
case <-ctx.Done():
	return ctx.Err()
case result := <-done:
	return result.Err
}
```

常见结果：

```go
context.Canceled
context.DeadlineExceeded
```

主动取消和超时在日志、指标、重试策略上可能不同：

```go
if errors.Is(err, context.DeadlineExceeded) {
	metrics.Timeouts.Add(1)
}
if errors.Is(err, context.Canceled) {
	metrics.Canceled.Add(1)
}
```

## 不要把 context 存进结构体

不推荐：

```go
type Service struct {
	ctx context.Context
}

func (s *Service) GetUser(id int64) (*User, error) {
	return s.repo.Get(s.ctx, id)
}
```

问题是：不同请求应该有不同的取消、超时、trace 信息。把 ctx 存进结构体会模糊生命周期。

推荐：

```go
type Service struct {
	repo *Repository
}

func (s *Service) GetUser(ctx context.Context, id int64) (*User, error) {
	return s.repo.Get(ctx, id)
}
```

少数长期后台组件可以持有用于整体生命周期的 context，但那应该是明确的设计，不是普通请求 ctx 的写法。

## context value 的边界

适合放请求范围元数据：

```go
type traceIDKey struct{}

func WithTraceID(ctx context.Context, id string) context.Context {
	return context.WithValue(ctx, traceIDKey{}, id)
}

func TraceID(ctx context.Context) (string, bool) {
	id, ok := ctx.Value(traceIDKey{}).(string)
	return id, ok
}
```

不适合放业务必填参数：

```go
func CreateOrder(ctx context.Context) error {
	userID := ctx.Value("user_id").(int64) // 不推荐
	_ = userID
	return nil
}
```

更好的签名：

```go
func CreateOrder(ctx context.Context, userID int64, req CreateOrderRequest) error {
	return nil
}
```

业务参数应该显式出现在函数签名里，调用方和测试才清楚。

## 面试追问

追问参考答案：[follow-ups.md](follow-ups.md)

- context 取消后 goroutine 会自动退出吗？
- 为什么创建 `WithTimeout` 后仍然要调用 `cancel()`？
- `context.Canceled` 和 `context.DeadlineExceeded` 有什么区别？
- 为什么不建议把 context 存进结构体？
- `context.Value` 适合放什么，不适合放什么？
- 如何写一个支持取消的 worker？
