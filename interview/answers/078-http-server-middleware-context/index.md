# 078. HTTP server

## 问题

Go HTTP server 中 middleware、request context 和超时应该怎么设计？

## 先给结论

Go HTTP server 的核心抽象是 `http.Handler`。middleware 本质上是接收一个 handler、返回一个新 handler 的包装函数。生产服务里，handler 要传递 `r.Context()` 给下游，server 要设置读写超时，middleware 要处理日志、恢复、认证、限流、trace 等横切逻辑。

## 1. Handler 和 HandlerFunc 的关系

`http.Handler` 只有一个方法：

```go
type Handler interface {
	ServeHTTP(ResponseWriter, *Request)
}
```

普通函数可以通过 `http.HandlerFunc` 适配成 Handler。

```go
func hello(w http.ResponseWriter, r *http.Request) {
	w.Write([]byte("hello"))
}

http.Handle("/hello", http.HandlerFunc(hello))
```

通常可以直接写：

```go
http.HandleFunc("/hello", hello)
```

理解这个接口，middleware 就很好理解。

## 2. middleware 是 Handler 包 Handler

一个日志 middleware：

```go
func logging(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		start := time.Now()
		next.ServeHTTP(w, r)
		log.Printf("%s %s %s", r.Method, r.URL.Path, time.Since(start))
	})
}
```

使用：

```go
handler := logging(http.HandlerFunc(hello))
http.ListenAndServe(":8080", handler)
```

多个 middleware 组合时，顺序很重要。

```go
handler := recoverPanic(logging(auth(router)))
```

外层先收到请求，内层处理业务，返回时再一层层回到外层。

## 3. middleware 写响应后不要继续调用 next

认证失败时应该直接返回。

```go
func auth(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		if r.Header.Get("Authorization") == "" {
			http.Error(w, "unauthorized", http.StatusUnauthorized)
			return
		}
		next.ServeHTTP(w, r)
	})
}
```

错误写法：

```go
if unauthorized {
	http.Error(w, "unauthorized", http.StatusUnauthorized)
}
next.ServeHTTP(w, r) // 可能继续写业务响应
```

这会导致重复写响应、状态码混乱。

## 4. handler 内要传递 request context

客户端断开、请求被取消、服务关闭时，`r.Context()` 会被取消。

```go
func getUser(w http.ResponseWriter, r *http.Request) {
	ctx := r.Context()

	user, err := repo.LoadUser(ctx, r.URL.Query().Get("id"))
	if err != nil {
		http.Error(w, err.Error(), 500)
		return
	}

	json.NewEncoder(w).Encode(user)
}
```

下游 DB、HTTP、RPC 都应该接收这个 ctx。

```go
row := db.QueryRowContext(ctx, "select name from users where id = ?", id)
```

不要在 handler 里启动后台 goroutine 后继续使用 `r.Context()` 做长生命周期任务。请求结束后它会被取消。

## 5. Server 级超时保护连接阶段

不要只写：

```go
http.ListenAndServe(":8080", handler)
```

更好的方式是显式配置 server：

```go
srv := &http.Server{
	Addr:              ":8080",
	Handler:           handler,
	ReadHeaderTimeout: 2 * time.Second,
	ReadTimeout:       5 * time.Second,
	WriteTimeout:      10 * time.Second,
	IdleTimeout:       60 * time.Second,
}

if err := srv.ListenAndServe(); err != nil && err != http.ErrServerClosed {
	return err
}
```

常见含义：

- `ReadHeaderTimeout`：限制读取请求头时间，防慢速头攻击。
- `ReadTimeout`：限制读取整个请求时间。
- `WriteTimeout`：限制写响应时间。
- `IdleTimeout`：限制 keep-alive 空闲连接时间。

长连接、SSE、WebSocket、流式响应要单独设计，不要盲目套普通 `WriteTimeout`。

## 6. 用 httptest 测 middleware

```go
func TestAuth(t *testing.T) {
	next := http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		w.WriteHeader(http.StatusOK)
	})

	req := httptest.NewRequest(http.MethodGet, "/", nil)
	rr := httptest.NewRecorder()

	auth(next).ServeHTTP(rr, req)

	if rr.Code != http.StatusUnauthorized {
		t.Fatalf("status = %d", rr.Code)
	}
}
```

middleware 不需要真的启动端口也能测试。

## 7. 面试时怎么答

可以这样回答：

- `http.Handler` 是核心抽象，`HandlerFunc` 让普通函数实现 Handler。
- middleware 是 `func(http.Handler) http.Handler`，通过包装 handler 实现横切逻辑。
- middleware 写了错误响应后要 return，不能继续调用 next。
- handler 里的下游调用要传递 `r.Context()`。
- server 要配置 ReadHeaderTimeout、ReadTimeout、WriteTimeout、IdleTimeout。
- 长连接和流式响应需要单独考虑 timeout。
- middleware 可以用 `httptest` 做单元测试。

## 面试追问

追问参考答案：[follow-ups.md](follow-ups.md)

- `http.HandlerFunc` 为什么能当 Handler 用？
- middleware 的执行顺序怎么判断？
- request context 什么时候会取消？
- server 的几个 timeout 分别保护什么？
- 如何测试 middleware 不继续调用 next？
