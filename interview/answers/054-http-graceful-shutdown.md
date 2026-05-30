# 054. HTTP 优雅关闭

## 问题

如何设计一个可优雅关闭的 Go HTTP 服务？

## 核心答案

优雅关闭需要处理系统信号，停止接收新请求，给存量请求一段时间完成，并在超时后退出。

Go 标准库 `http.Server.Shutdown(ctx)` 会关闭监听器并等待活跃连接结束。业务 goroutine 也应该监听 context 退出。

## 代码示例

```go
srv := &http.Server{Addr: ":8080", Handler: mux}

go srv.ListenAndServe()

ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
defer cancel()
_ = srv.Shutdown(ctx)
```

## 面试追问

- `Shutdown` 和 `Close` 有什么区别？
- 长连接和后台任务如何处理？
