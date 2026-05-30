# 053. netpoller

## 问题

Go netpoller 的作用是什么？为什么大量连接不需要大量线程？

## 核心答案

netpoller 是 Go runtime 对网络 I/O 多路复用的封装。网络 fd 等待可读可写时，goroutine 可以挂起，底层线程去执行其他 goroutine。

当 fd 就绪时，runtime 再把对应 goroutine 放回可运行队列。因此大量网络连接不需要一连接一线程。

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

- 阻塞网络 I/O 为什么不会阻塞整个线程池？
- netpoller 和调度器如何协作？
