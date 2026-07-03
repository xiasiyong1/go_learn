# 054. HTTP 优雅关闭

## 问题

如何设计一个可优雅关闭的 Go HTTP 服务？

## 先给结论

HTTP 优雅关闭的目标是停止接收新请求，给正在处理的请求和后台任务有限时间完成，然后释放资源。关键是信号处理、超时、连接状态和任务生命周期。

## 深入理解

### 1. 这道题真正考察什么

- 是否知道 `Server.Shutdown` 和 `Server.Close` 的区别。
- 是否能处理 SIGTERM/SIGINT 信号。
- 是否能让 readiness 先下线，再等待流量摘除。
- 是否能处理长连接、websocket、后台 goroutine 和数据库连接。

### 2. 底层机制要讲清楚

- `Shutdown(ctx)` 会关闭监听器，关闭空闲连接，并等待活跃连接返回或 ctx 超时。
- `Close()` 会立即关闭连接，更偏强制终止。
- HTTP server 只知道请求连接，不知道你额外启动的后台任务。
- 优雅关闭需要和负载均衡、容器编排、健康检查协同。

### 3. 工程实践怎么取舍

- 收到终止信号后先拒绝新流量或标记 not ready。
- 调用 `Shutdown` 时传入有上限的 context。
- 请求 handler 内部继续传递 context，让下游调用能取消。
- 后台任务使用独立的生命周期管理，关闭时 cancel 并 Wait。

### 4. 常见误区

- 只调用 Shutdown，忘记等待后台 goroutine。
- 没有超时时间，关闭过程可能无限等待。
- readiness 没有先下线，新请求还在进入。
- 长连接协议不响应 Shutdown，需要单独管理。

## 如何验证理解

- 用集成测试启动 server，发起慢请求后触发 Shutdown。
- 用日志和指标观察关闭期间新请求、活跃请求和超时数量。
- 在本地发送 SIGTERM 验证进程退出码和资源释放。

## 代码示例

```go
srv := &http.Server{Addr: ":8080", Handler: mux}

go srv.ListenAndServe()

ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
defer cancel()
_ = srv.Shutdown(ctx)
```

## 面试追问

追问参考答案：[follow-ups.md](follow-ups.md)

- 如果继续追问“HTTP 优雅关闭”的底层机制，应该讲到哪些层次？
- 这个知识点在真实项目里应该如何取舍？
- 最容易踩的坑是什么？为什么这些坑不是背结论就能避免的？
- 如何用测试、工具或 profiling 验证自己的判断？
- 当数据量、并发量或团队规模变大后，这个问题会怎样升级？
