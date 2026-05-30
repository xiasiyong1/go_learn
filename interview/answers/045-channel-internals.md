# 045. channel 底层实现

## 问题

channel 底层大致如何实现？阻塞和唤醒是怎么发生的？

## 核心答案

channel 运行时结构中包含缓冲区、发送等待队列、接收等待队列和锁。

发送时如果有等待接收者，可以直接交付；否则如果缓冲区未满就写入缓冲区；再否则当前 goroutine 会挂起进入发送队列。接收过程类似。

关闭 channel 会唤醒等待的接收者和发送者，等待发送者会 panic。

## 代码示例

```go
ch := make(chan int, 1)
ch <- 1
fmt.Println(<-ch)
```

## 面试追问

- channel 为什么不是无锁结构？
- nil channel 读写为什么会永久阻塞？
