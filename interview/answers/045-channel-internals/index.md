# 045. channel 底层实现

## 问题

channel 底层大致如何实现？阻塞和唤醒是怎么发生的？

## 先给结论

channel 底层可理解为带锁的队列加发送/接收等待队列。它负责数据传递、阻塞唤醒和内存同步，但不是无锁结构，也不是所有并发问题的最佳答案。

## 深入理解

### 1. 这道题真正考察什么

- 是否知道 channel 内部需要维护缓冲区、等待发送队列和等待接收队列。
- 是否能解释无缓冲和有缓冲的阻塞唤醒差异。
- 是否知道 nil channel 为什么永久阻塞。
- 是否能说明 close 如何唤醒接收者。

### 2. 底层机制要讲清楚

- channel 内部结构大致包含锁、元素类型、缓冲队列、sendq、recvq 和关闭状态。
- 无缓冲 channel 发送和接收直接配对交接数据。
- 有缓冲 channel 发送先尝试进入缓冲，接收先尝试从缓冲取出。
- 没有可配对操作时，当前 goroutine 会挂到等待队列，等待后续唤醒。

### 3. 工程实践怎么取舍

- 用 channel 表达所有权转移、任务队列和事件通知。
- 频繁竞争的简单计数或状态保护，mutex/atomic 可能更直接。
- 关闭 channel 时明确发送方所有权，避免多方 close。
- 避免在高频路径创建大量短生命周期 channel。

### 4. 常见误区

- 认为 channel 无锁，所以一定比 mutex 快。
- nil channel 在 select 里无意禁用 case，导致逻辑卡死。
- 关闭 channel 后仍有发送者，触发 panic。
- 缓冲过大，让队列堆积变成内存和延迟问题。

## 如何验证理解

- 用 block profile 查看 goroutine 阻塞在 send 还是 receive。
- 用 benchmark 对比 channel、mutex、atomic 在具体场景的成本。
- 写并发测试覆盖 close、cancel 和慢消费者。

## 代码示例

```go
ch := make(chan int, 1)
ch <- 1
fmt.Println(<-ch)
```

## 面试追问

追问参考答案：[follow-ups.md](follow-ups.md)

- 如果继续追问“channel 底层实现”的底层机制，应该讲到哪些层次？
- 这个知识点在真实项目里应该如何取舍？
- 最容易踩的坑是什么？为什么这些坑不是背结论就能避免的？
- 如何用测试、工具或 profiling 验证自己的判断？
- 当数据量、并发量或团队规模变大后，这个问题会怎样升级？
