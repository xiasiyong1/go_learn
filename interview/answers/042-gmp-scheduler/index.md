# 042. GMP 调度器

## 问题

Go 调度器 GMP 模型是什么？

## 核心答案

G 表示 goroutine，M 表示操作系统线程，P 表示处理器上下文。M 需要绑定 P 才能执行 Go 代码，P 持有本地运行队列。

调度器会把大量 goroutine 映射到较少的系统线程上执行，并通过工作窃取、网络轮询和抢占调度提高吞吐和公平性。

## 代码示例

```go
runtime.GOMAXPROCS(runtime.NumCPU())
```

## 面试追问

追问参考答案：[follow-ups.md](follow-ups.md)

- GOMAXPROCS 控制什么？
- goroutine 阻塞系统调用时调度器如何处理？
