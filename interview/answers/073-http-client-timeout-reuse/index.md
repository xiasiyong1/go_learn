# 073. HTTP client

## 问题

Go HTTP client 的超时、连接复用和响应体关闭有哪些坑？

## 先给结论

生产代码里不要每次请求都新建 `http.Client`，也不要直接依赖没有超时的默认配置。`http.Client` 应该复用，超时要分层设置，`resp.Body` 必须关闭；如果要复用连接，通常还要读完响应体或确保 body 被正确丢弃。

## 1. 默认 client 没有整体超时

下面的代码能跑，但下游一直不返回时，请求可能长时间挂住。

```go
resp, err := http.Get("https://example.com")
if err != nil {
	return err
}
defer resp.Body.Close()
```

更稳的写法是配置 client：

```go
client := &http.Client{
	Timeout: 3 * time.Second,
}

resp, err := client.Get("https://example.com")
if err != nil {
	return err
}
defer resp.Body.Close()
```

`Client.Timeout` 是从发起请求到读取响应体的整体上限，适合简单场景。复杂场景还要配置 Transport 的拨号、TLS、响应头等阶段超时。

## 2. client 应该复用，不应该每次创建

错误写法：

```go
func fetch(url string) error {
	client := &http.Client{Timeout: 3 * time.Second}
	resp, err := client.Get(url)
	if err != nil {
		return err
	}
	defer resp.Body.Close()
	return nil
}
```

这样会让连接池难以复用，增加 TCP/TLS 握手成本。推荐把 client 作为依赖传入或全局复用：

```go
type API struct {
	client *http.Client
	base   string
}

func NewAPI(base string) *API {
	return &API{
		base: base,
		client: &http.Client{
			Timeout: 3 * time.Second,
		},
	}
}
```

高并发调用下游时，应该显式配置连接池。

```go
transport := &http.Transport{
	MaxIdleConns:        100,
	MaxIdleConnsPerHost: 20,
	IdleConnTimeout:     90 * time.Second,
}

client := &http.Client{
	Transport: transport,
	Timeout:   3 * time.Second,
}
```

## 3. `resp.Body` 必须关闭

不关闭 body 会导致连接和资源不能及时释放。

```go
resp, err := client.Get(url)
if err != nil {
	return err
}
defer resp.Body.Close()

body, err := io.ReadAll(resp.Body)
if err != nil {
	return err
}
_ = body
```

如果只关心状态码，也要关闭 body。为了更好复用连接，可以丢弃 body 后关闭：

```go
resp, err := client.Get(url)
if err != nil {
	return err
}
defer resp.Body.Close()

_, _ = io.Copy(io.Discard, resp.Body)
```

是否必须读完才能复用，和协议、Transport 状态有关。面试里保守回答是：关闭必须做；需要连接复用时，应确保 body 被读完或丢弃到 EOF。

## 4. context timeout 和 client timeout 怎么配合

请求级别更推荐从调用链传入 context。

```go
ctx, cancel := context.WithTimeout(parent, 500*time.Millisecond)
defer cancel()

req, err := http.NewRequestWithContext(ctx, http.MethodGet, url, nil)
if err != nil {
	return err
}

resp, err := client.Do(req)
if err != nil {
	return err
}
defer resp.Body.Close()
```

`client.Timeout` 是 client 统一上限，`context` 是单次请求或调用链预算。真实服务中通常两者都会有：client 兜底，请求 context 表达业务 deadline。

## 5. Transport 分阶段超时

下游问题可能发生在不同阶段：连接慢、TLS 慢、响应头慢、body 慢。

```go
transport := &http.Transport{
	DialContext: (&net.Dialer{
		Timeout:   500 * time.Millisecond,
		KeepAlive: 30 * time.Second,
	}).DialContext,
	TLSHandshakeTimeout:   500 * time.Millisecond,
	ResponseHeaderTimeout: 1 * time.Second,
	ExpectContinueTimeout: 1 * time.Second,
}

client := &http.Client{
	Transport: transport,
	Timeout:   3 * time.Second,
}
```

面试时不需要背所有字段，但要知道“HTTP 超时不是一个点”，它覆盖多个阶段。

## 6. 用 httptest 验证超时

可以写一个慢响应服务：

```go
server := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
	time.Sleep(200 * time.Millisecond)
	w.WriteHeader(http.StatusOK)
}))
defer server.Close()

client := &http.Client{Timeout: 50 * time.Millisecond}

_, err := client.Get(server.URL)
fmt.Println(err != nil) // true
```

这种测试比只看代码更可靠。

## 7. 面试时怎么答

可以这样回答：

- `http.Client` 要复用，因为它内部通过 Transport 复用连接。
- 默认 client 没有整体超时，生产代码要设置 timeout。
- `resp.Body` 必须关闭；为了连接复用，通常还要读完或丢弃 body。
- context timeout 表示单次请求预算，client timeout 是 client 层兜底。
- 高并发下要配置连接池，如 `MaxIdleConnsPerHost`。
- 复杂问题要区分拨号、TLS、响应头、响应体等阶段超时。

## 面试追问

追问参考答案：[follow-ups.md](follow-ups.md)

- 为什么 `http.Client` 要复用？
- 只调用 `resp.Body.Close()` 够不够？
- `Client.Timeout` 和 `context.WithTimeout` 有什么区别？
- 如何用 `httptest` 验证超时和取消？
- 高并发调用下游时连接池怎么配置？
