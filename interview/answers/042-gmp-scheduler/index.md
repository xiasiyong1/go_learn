# 042. GMP 调度器

## 问题

Go 调度器 GMP 模型是什么？

## 先给结论

GMP 是 Go 调度器的核心模型：G 是 goroutine，M 是 OS 线程，P 是处理器资源和本地运行队列。它解释了 Go 如何用较少线程调度大量 goroutine。

## 深入理解

### 1. 这道题真正考察什么

- 是否能说清 G、M、P 分别代表什么。
- 是否知道 GOMAXPROCS 控制可同时执行 Go 代码的 P 数量。
- 是否能解释系统调用、网络阻塞和抢占对调度的影响。
- 是否知道 work stealing 如何提升负载均衡。

### 2. 底层机制要讲清楚

- G 保存 goroutine 的栈、状态和执行信息。
- M 是真正执行代码的系统线程。
- P 持有本地运行队列和执行 Go 代码所需资源，M 必须绑定 P 才能执行 Go 代码。
- 当某个 P 本地队列空了，会从全局队列或其他 P 窃取 G。

### 3. 工程实践怎么取舍

- CPU 密集任务受 GOMAXPROCS 影响明显。
- 阻塞系统调用会让运行时把 P 交给其他 M，避免全部停住。
- 网络 I/O 通常结合 netpoller，避免每个连接长期占用线程。
- 不要用无限忙循环占住 P，应让出调度或使用阻塞原语。

### 4. 常见误区

- 以为 goroutine 数量等于线程数量。
- 把 GOMAXPROCS 当成 goroutine 上限。
- CPU 忙循环没有函数调用或阻塞点，影响抢占和整体延迟。
- 启动海量 goroutine 但没有资源上限，导致内存和调度压力。

## 如何验证理解

- 用 `GOMAXPROCS` 改变 CPU 密集 benchmark，观察吞吐变化。
- 用 `go tool trace` 查看 goroutine 状态和调度延迟。
- 用 pprof goroutine profile 观察大量 goroutine 是否阻塞在同类位置。

## 代码示例

```go
runtime.GOMAXPROCS(runtime.NumCPU())
```

## 面试追问

追问参考答案：[follow-ups.md](follow-ups.md)

- 如果继续追问“GMP 调度器”的底层机制，应该讲到哪些层次？
- 这个知识点在真实项目里应该如何取舍？
- 最容易踩的坑是什么？为什么这些坑不是背结论就能避免的？
- 如何用测试、工具或 profiling 验证自己的判断？
- 当数据量、并发量或团队规模变大后，这个问题会怎样升级？
