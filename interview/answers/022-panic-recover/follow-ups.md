# 022. panic 和 recover - 面试追问

## 1. panic 发生后 defer 的执行顺序是什么？

panic 后，当前函数停止正常执行，开始按后进先出顺序执行 defer。

```go
func f() {
	defer fmt.Println("first")
	defer fmt.Println("second")

	panic("boom")
}
```

输出：

```text
second
first
panic: boom
```

如果某个 defer 成功 recover，panic 展开会停止，函数返回。

```go
func f() {
	defer func() {
		if r := recover(); r != nil {
			fmt.Println("recovered:", r)
		}
	}()
	panic("boom")
}
```

## 2. recover 为什么必须在 defer 函数中直接调用？

`recover` 只有在 panic 展开过程中，由 defer 函数直接调用才有效。

有效：

```go
defer func() {
	if r := recover(); r != nil {
		fmt.Println(r)
	}
}()
```

无效：

```go
func doRecover() {
	recover()
}

defer func() {
	doRecover()
}()
```

第二种写法里，`recover` 不是 defer 函数直接调用，不能恢复 panic。

面试时可以说：recover 的有效位置非常窄，不是到处调用都能捕获异常。

## 3. recover 能不能捕获另一个 goroutine 的 panic？

不能。

```go
func main() {
	defer func() {
		if r := recover(); r != nil {
			fmt.Println("main recovered:", r)
		}
	}()

	go func() {
		panic("worker panic")
	}()

	time.Sleep(time.Second)
}
```

`main` 的 recover 捕获不到 worker goroutine 的 panic。

正确做法是在 goroutine 内部加 recover：

```go
go func() {
	defer func() {
		if r := recover(); r != nil {
			slog.Error("worker panic", "panic", r, "stack", string(debug.Stack()))
		}
	}()

	doWork()
}()
```

不同 goroutine 有独立调用栈，recover 不能跨栈。

## 4. HTTP 服务里应该在哪里 recover？

应该在请求边界，比如 middleware。

```go
func Recover(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		defer func() {
			if v := recover(); v != nil {
				slog.Error("panic",
					"panic", v,
					"path", r.URL.Path,
					"stack", string(debug.Stack()),
				)
				http.Error(w, "internal server error", http.StatusInternalServerError)
			}
		}()

		next.ServeHTTP(w, r)
	})
}
```

这样能防止单个请求 panic 直接结束进程。

但业务层仍然应该返回 error：

```go
func CreateUser(req CreateUserRequest) error {
	if req.Name == "" {
		return fmt.Errorf("name is required")
	}
	return nil
}
```

不要把 recover 当成业务错误处理器。

## 5. recover 后为什么要记录 stack？

panic 通常说明出现了未预期路径。只有 panic 值往往不够定位问题。

不推荐：

```go
defer func() {
	_ = recover()
}()
```

推荐：

```go
defer func() {
	if r := recover(); r != nil {
		slog.Error("panic recovered",
			"panic", r,
			"stack", string(debug.Stack()),
		)
	}
}()
```

堆栈能告诉你 panic 发生在哪一行、调用链是什么，这比只看 `"panic: index out of range"` 有用得多。

## 6. panic 和 error 的边界应该怎么划分？

普通、可预期、调用方能处理的失败，用 error。

```go
func OpenConfig(path string) ([]byte, error) {
	b, err := os.ReadFile(path)
	if err != nil {
		return nil, fmt.Errorf("read config: %w", err)
	}
	return b, nil
}
```

不变量被破坏、程序员错误、无法继续的状态，可以考虑 panic。

```go
func MustRegister(name string, h Handler) {
	if name == "" || h == nil {
		panic("invalid handler registration")
	}
	registry[name] = h
}
```

经验是：库函数不要轻易 panic 逼调用方 recover；服务边界可以 recover 兜底，但业务流程仍然应该用 error 表达失败。
