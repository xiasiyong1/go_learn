# 053. netpoller

## 问题

Go netpoller 的作用是什么？为什么大量连接不需要大量线程？

## 先给结论

netpoller 让 Go 可以用非阻塞网络 I/O 管理大量连接，而不必让每个阻塞网络操作长期占用一个 OS 线程。它与调度器协作唤醒等待 I/O 的 goroutine。

## 深入理解

### 1. 这道题真正考察什么

- 是否知道网络阻塞和普通系统调用阻塞在调度上有所不同。
- 是否能解释为什么大量连接不等于大量线程。
- 是否理解 goroutine 会在等待 I/O 时挂起，I/O 就绪后被重新调度。
- 是否能说明 deadline、context 和连接关闭如何影响网络等待。

### 2. 底层机制要讲清楚

- Go 网络库把 fd 设置为非阻塞，并注册到平台 I/O 多路复用机制。
- goroutine 发起读写但 fd 未就绪时，会被挂起，线程可以执行其他 goroutine。
- netpoller 发现 fd 就绪后，把对应 goroutine 放回可运行队列。
- deadline 通过 runtime timer 与 netpoller 协作，让等待能按时超时。

### 3. 工程实践怎么取舍

- 为网络读写设置 deadline 或通过 context 控制请求生命周期。
- 不要让单连接 goroutine 无退出条件地等待。
- 大量连接服务要关注 fd 上限、内存、心跳、慢连接和 backpressure。
- CPU 密集处理不要直接堵在网络 goroutine 中，应限制并发或异步处理。

### 4. 常见误区

- 认为每个连接都会占一个线程，所以 Go 不能支撑大量连接。
- 没有设置 read deadline，慢连接长期占用 goroutine 和资源。
- 只优化 netpoller，忽略业务处理 CPU 或下游瓶颈。
- 把连接关闭和请求取消路径设计得不完整，导致 goroutine 泄漏。

## 如何验证理解

- 用 pprof goroutine profile 看连接 goroutine 阻塞在读、写还是业务逻辑。
- 用 trace 观察网络等待和调度唤醒。
- 压测慢连接和大量空闲连接，验证 deadline 与资源上限。

## 代码示例

```go
ln, _ := net.Listen("tcp", ":8080")
for {
	conn, _ := ln.Accept()
	go handle(conn)
}
```

## 面试追问

追问参考答案：[follow-ups.md](follow-ups.md)

- 如果继续追问“netpoller”的底层机制，应该讲到哪些层次？
- 这个知识点在真实项目里应该如何取舍？
- 最容易踩的坑是什么？为什么这些坑不是背结论就能避免的？
- 如何用测试、工具或 profiling 验证自己的判断？
- 当数据量、并发量或团队规模变大后，这个问题会怎样升级？
