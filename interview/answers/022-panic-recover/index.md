# 022. panic 和 recover

## 问题

`panic` 和 `recover` 的使用边界是什么？

## 先给结论

`panic` 表示当前调用栈无法继续正常执行。发生 panic 后，当前 goroutine 会停止正常流程，开始执行已经注册的 `defer`。

`recover` 只能在同一个 goroutine 的 defer 函数中直接调用，才能捕获 panic。

普通业务失败应该返回 `error`，不要用 `panic` 当异常机制。`panic/recover` 更适合：

- 程序员错误或不变量被破坏。
- 初始化阶段无法继续的严重错误。
- HTTP middleware、任务执行器、goroutine wrapper 这种边界兜底。

## panic 会执行 defer

```go
func f() {
	defer fmt.Println("defer 1")
	defer fmt.Println("defer 2")

	panic("boom")
}
```

输出顺序：

```text
defer 2
defer 1
panic: boom
```

多个 `defer` 仍然按后进先出执行。

## recover 必须在 defer 中直接调用

正确写法：

```go
func safe() {
	defer func() {
		if r := recover(); r != nil {
			fmt.Println("recovered:", r)
		}
	}()

	panic("boom")
}
```

错误写法：

```go
func recoverOutside() {
	if r := recover(); r != nil {
		fmt.Println(r)
	}
}

func bad() {
	recoverOutside()
	panic("boom")
}
```

`recoverOutside` 不是在 panic 展开过程中的 defer 里直接调用，不能恢复。

还要注意，不要把 recover 藏到普通函数里再间接调用：

```go
func doRecover() {
	recover()
}

func bad() {
	defer func() {
		doRecover() // 无效
	}()
	panic("boom")
}
```

要直接在 defer 函数里调用 `recover()`。

## recover 不能跨 goroutine

```go
func main() {
	defer func() {
		if r := recover(); r != nil {
			fmt.Println("recovered:", r)
		}
	}()

	go func() {
		panic("worker failed")
	}()

	time.Sleep(time.Second)
}
```

`main` goroutine 里的 recover 捕获不到子 goroutine 的 panic。不同 goroutine 有不同调用栈。

如果启动 goroutine，需要在 goroutine 内部兜底：

```go
func Go(fn func()) {
	go func() {
		defer func() {
			if r := recover(); r != nil {
				slog.Error("goroutine panic", "panic", r, "stack", string(debug.Stack()))
			}
		}()
		fn()
	}()
}
```

长期运行服务里，后台 goroutine 没有 recover，panic 可能直接让进程退出。

## recover 后不要静默吞掉

不推荐：

```go
defer func() {
	_ = recover()
}()
```

这样问题消失在日志里，排查时没有上下文和堆栈。

推荐至少记录 panic 值和堆栈：

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

如果函数签名允许返回错误，可以把 panic 转成 error：

```go
func SafeCall(fn func()) (err error) {
	defer func() {
		if r := recover(); r != nil {
			err = fmt.Errorf("panic: %v", r)
		}
	}()

	fn()
	return nil
}
```

## HTTP middleware 里的 recover

```go
func Recover(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		defer func() {
			if v := recover(); v != nil {
				slog.Error("handler panic",
					"panic", v,
					"stack", string(debug.Stack()),
				)
				http.Error(w, "internal server error", http.StatusInternalServerError)
			}
		}()

		next.ServeHTTP(w, r)
	})
}
```

这种 recover 放在请求边界是合理的：避免一个请求把整个服务进程打崩，并且记录堆栈。

但业务函数内部仍然应该返回 error，而不是靠 middleware recover 做正常控制流。

## 不是所有崩溃都能 recover

有些 runtime fatal error 不能用普通 recover 处理，例如并发 map 写常见的 fatal error。

```go
// fatal error: concurrent map writes
```

这类问题要修并发安全根因，不能指望 recover 兜底。

## 面试追问

追问参考答案：[follow-ups.md](follow-ups.md)

- panic 发生后 defer 的执行顺序是什么？
- recover 为什么必须在 defer 函数中直接调用？
- recover 能不能捕获另一个 goroutine 的 panic？
- HTTP 服务里应该在哪里 recover？
- recover 后为什么要记录 stack？
- panic 和 error 的边界应该怎么划分？
