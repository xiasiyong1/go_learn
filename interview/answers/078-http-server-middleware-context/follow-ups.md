# 078. HTTP server - 面试追问

## 1. `http.HandlerFunc` 为什么能当 Handler 用？

因为标准库给函数类型定义了 `ServeHTTP` 方法。

```go
type HandlerFunc func(ResponseWriter, *Request)

func (f HandlerFunc) ServeHTTP(w ResponseWriter, r *Request) {
	f(w, r)
}
```

所以普通函数可以被转换成 Handler。

```go
func hello(w http.ResponseWriter, r *http.Request) {
	w.Write([]byte("hello"))
}

var h http.Handler = http.HandlerFunc(hello)
```

这也是 Go 里“函数适配接口”的经典例子。

## 2. middleware 的执行顺序怎么判断？

看包装顺序。

```go
handler := A(B(C(final)))
```

请求进入顺序是 A -> B -> C -> final；响应返回时是 final -> C -> B -> A。

代码示例：

```go
func named(name string, next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		fmt.Println("before", name)
		next.ServeHTTP(w, r)
		fmt.Println("after", name)
	})
}
```

组合：

```go
h := named("A", named("B", http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
	fmt.Println("handler")
})))
```

输出：

```text
before A
before B
handler
after B
after A
```

## 3. request context 什么时候会取消？

常见场景：

- 客户端连接断开。
- HTTP/2 请求被取消。
- server 开始关闭。
- handler 返回后，request context 生命周期结束。

handler 内要把 `r.Context()` 传给下游：

```go
func handler(w http.ResponseWriter, r *http.Request) {
	req, _ := http.NewRequestWithContext(r.Context(), http.MethodGet, downstream, nil)
	resp, err := client.Do(req)
	if err != nil {
		http.Error(w, err.Error(), 500)
		return
	}
	defer resp.Body.Close()
}
```

不要把 `r.Context()` 存起来给后台任务长期使用。如果确实要在请求结束后继续跑任务，应该创建独立生命周期的 context，并明确任务归属。

## 4. server 的几个 timeout 分别保护什么？

```go
srv := &http.Server{
	ReadHeaderTimeout: 2 * time.Second,
	ReadTimeout:       5 * time.Second,
	WriteTimeout:      10 * time.Second,
	IdleTimeout:       60 * time.Second,
}
```

含义：

- `ReadHeaderTimeout`：限制请求头读取时间，防慢速头攻击。
- `ReadTimeout`：限制读取整个请求，包括 body。
- `WriteTimeout`：限制响应写出时间。
- `IdleTimeout`：keep-alive 连接空闲多久关闭。

注意：大文件上传、下载、SSE、WebSocket 对 timeout 要单独设计。普通短请求配置不能直接套到所有场景。

## 5. 如何测试 middleware 不继续调用 next？

可以让 next 修改一个变量。如果鉴权失败后变量仍然是 false，说明没有继续调用。

```go
func TestAuthStopsChain(t *testing.T) {
	called := false
	next := http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		called = true
	})

	req := httptest.NewRequest(http.MethodGet, "/", nil)
	rr := httptest.NewRecorder()

	auth(next).ServeHTTP(rr, req)

	if called {
		t.Fatal("next should not be called")
	}
	if rr.Code != http.StatusUnauthorized {
		t.Fatalf("status = %d", rr.Code)
	}
}
```

这个测试比只断言状态码更严格，因为它能发现“写了 401 但仍继续执行 handler”的问题。
