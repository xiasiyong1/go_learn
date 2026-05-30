# 028. goroutine 和 channel 基础

## 问题

goroutine 和 channel 的基本通信模型是什么？

## 核心答案

goroutine 是由 Go runtime 调度的轻量级执行单元。channel 用于 goroutine 之间通信和同步。

Go 的并发理念是通过通信共享内存，而不是只靠共享内存通信。但这不是说不能用锁，实际工程中 channel 和锁要按场景选择。

## 代码示例

```go
ch := make(chan int)
go func() {
	ch <- 1
}()
fmt.Println(<-ch)
```

## 面试追问

- goroutine 和系统线程有什么区别？
- channel 是否一定比 mutex 更好？
