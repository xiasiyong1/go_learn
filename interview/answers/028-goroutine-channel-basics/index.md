# 028. goroutine 和 channel 基础

## 问题

goroutine 和 channel 的基本通信模型是什么？

## 先给结论

goroutine 是 Go 调度器管理的轻量并发执行单元，channel 是类型安全的通信和同步原语。关键不是“channel 比锁高级”，而是选择合适的共享状态模型。

## 深入理解

### 1. 这道题真正考察什么

- 是否知道 goroutine 不是系统线程，但最终运行在线程上。
- 是否理解 channel 既能传数据，也能建立同步关系。
- 是否能区分通过通信共享内存和通过锁保护共享内存。
- 是否能设计 goroutine 的退出和错误传播。

### 2. 底层机制要讲清楚

- Go 调度器把大量 goroutine 映射到较少 OS 线程上执行。
- 无缓冲 channel 发送和接收同步完成；有缓冲 channel 在缓冲未满或非空时可异步推进。
- channel send、receive、close 都和内存可见性有关。
- channel 本身不解决所有并发问题，错误关闭、泄漏、缓冲过大都可能引入问题。

### 3. 工程实践怎么取舍

- 数据所有权清晰、流水线式处理适合 channel。
- 多个 goroutine 频繁读写同一内存结构，mutex 可能更简单高效。
- 启动 goroutine 必须明确退出条件和结果处理路径。
- 不要把 channel 当队列无限堆积，容量和消费速度要匹配。

### 4. 常见误区

- 只启动 goroutine 不等待，主函数退出后任务直接消失。
- 发送方和接收方数量不匹配导致阻塞。
- 把 channel 关闭权交给多个发送方，导致 panic。
- 为了避免锁滥用 channel，反而让控制流更复杂。

## 如何验证理解

- 用 race detector 检查共享变量是否被安全访问。
- 用 pprof goroutine profile 查看是否有 channel 阻塞堆栈。
- 为并发函数写取消、超时和错误返回测试。

## 代码示例

```go
ch := make(chan int)
go func() {
	ch <- 1
}()
fmt.Println(<-ch)
```

## 面试追问

追问参考答案：[follow-ups.md](follow-ups.md)

- 如果继续追问“goroutine 和 channel 基础”的底层机制，应该讲到哪些层次？
- 这个知识点在真实项目里应该如何取舍？
- 最容易踩的坑是什么？为什么这些坑不是背结论就能避免的？
- 如何用测试、工具或 profiling 验证自己的判断？
- 当数据量、并发量或团队规模变大后，这个问题会怎样升级？
