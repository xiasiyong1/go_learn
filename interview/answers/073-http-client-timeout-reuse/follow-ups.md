# 073. HTTP client - 面试追问

## 1. 为什么 `http.Client` 要复用？

`http.Client` 通过内部的 `Transport` 维护连接池。复用 client 才能复用 TCP/TLS 连接。

不推荐每次请求创建：

```go
func bad(url string) error {
	client := &http.Client{Timeout: time.Second}
	resp, err := client.Get(url)
	if err != nil {
		return err
	}
	defer resp.Body.Close()
	return nil
}
```

推荐注入复用：

```go
type Client struct {
	http *http.Client
}

func NewClient() *Client {
	return &Client{
		http: &http.Client{Timeout: time.Second},
	}
}
```

复用不是为了少创建一个结构体，而是为了保留连接池、减少握手和 fd 压力。

## 2. 只调用 `resp.Body.Close()` 够不够？

关闭是必须的，但如果希望连接复用，通常还要让响应体被读到 EOF 或丢弃。

```go
resp, err := client.Get(url)
if err != nil {
	return err
}
defer resp.Body.Close()

_, _ = io.Copy(io.Discard, resp.Body)
```

如果响应体很大，不应该无脑读完，而要按业务限制处理：

```go
limited := io.LimitReader(resp.Body, 1<<20)
data, err := io.ReadAll(limited)
```

所以回答要有边界：关闭一定要做；读完有利于连接复用；大 body 要配合大小限制和业务策略。

## 3. `Client.Timeout` 和 `context.WithTimeout` 有什么区别？

`Client.Timeout` 是这个 client 发出的请求的整体上限。

```go
client := &http.Client{Timeout: 2 * time.Second}
```

context timeout 是某一次请求的调用链预算。

```go
ctx, cancel := context.WithTimeout(parent, 300*time.Millisecond)
defer cancel()

req, _ := http.NewRequestWithContext(ctx, http.MethodGet, url, nil)
resp, err := client.Do(req)
```

在服务里，下游调用应继承上游请求 context：

```go
func handler(w http.ResponseWriter, r *http.Request) {
	req, _ := http.NewRequestWithContext(r.Context(), http.MethodGet, downstream, nil)
	_, _ = client.Do(req)
}
```

这样客户端断开、请求超时、服务关闭时，下游请求能一起取消。

## 4. 如何用 `httptest` 验证超时和取消？

构造一个慢服务：

```go
srv := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
	time.Sleep(200 * time.Millisecond)
	w.WriteHeader(http.StatusOK)
}))
defer srv.Close()

client := &http.Client{Timeout: 50 * time.Millisecond}

_, err := client.Get(srv.URL)
fmt.Println(err != nil) // true
```

验证 context 取消：

```go
ctx, cancel := context.WithCancel(context.Background())
cancel()

req, _ := http.NewRequestWithContext(ctx, http.MethodGet, srv.URL, nil)
_, err := client.Do(req)
fmt.Println(errors.Is(err, context.Canceled)) // 不一定直接 Is，但 err 会表示取消
```

测试重点不是错误字符串，而是请求是否按预期时间返回、服务侧是否观察到 `r.Context().Done()`。

## 5. 高并发调用下游时连接池怎么配置？

需要按下游容量和本服务并发配置 Transport。

```go
tr := &http.Transport{
	MaxIdleConns:        200,
	MaxIdleConnsPerHost: 50,
	MaxConnsPerHost:     100,
	IdleConnTimeout:     90 * time.Second,
}

client := &http.Client{
	Transport: tr,
	Timeout:   2 * time.Second,
}
```

配置不是越大越好。太小会排队，太大会把下游打满。

真实服务要结合指标看：

- 当前连接数和空闲连接数。
- 请求等待连接的时间。
- 下游 P95/P99 延迟和错误率。
- 本服务 goroutine 数和超时率。
