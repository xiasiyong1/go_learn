# 050. benchmark、pprof 和 trace

## 问题

如何用 benchmark、pprof 和 trace 定位性能问题？

## 先给结论

benchmark、pprof 和 trace 分别回答不同问题：代码片段有多快、资源消耗在哪、运行时调度和阻塞发生了什么。优化前必须先测量。

## 深入理解

### 1. 这道题真正考察什么

- 是否知道 benchmark 用于稳定比较特定代码路径。
- 是否能区分 CPU、heap、goroutine、mutex、block profile。
- 是否知道 pprof 的 flat 和 cum 含义。
- 是否能说明 trace 更适合看调度、网络、GC 和阻塞时间线。

### 2. 底层机制要讲清楚

- benchmark 通过多次迭代降低噪声，但仍受机器、编译器和输入数据影响。
- CPU profile 采样正在消耗 CPU 的栈，heap profile 展示分配或存活内存。
- flat 是函数自身消耗，cum 是函数及其调用链累计消耗。
- trace 记录事件时间线，可以看到 goroutine 何时阻塞、唤醒、运行。

### 3. 工程实践怎么取舍

- 先明确优化目标：吞吐、延迟、内存、分配、锁竞争还是启动时间。
- 用生产相似数据和负载测量，避免微基准误导。
- 一次只改一个变量，保留基线。
- 优化后用测试和 benchmark 同时验证正确性和收益。

### 4. 常见误区

- 看到 pprof 顶部函数就直接改，没理解调用路径。
- 用 race 版本或 debug 环境数据判断生产性能。
- benchmark 被编译器优化掉，结果不可信。
- 只看平均值，忽略尾延迟和抖动。

## 如何验证理解

- 用 `go test -bench . -benchmem` 看微基准和分配。
- 用 `go test -cpuprofile` 或线上 pprof 抓 CPU 数据。
- 用 `go tool pprof` 的 top、list、web 定位路径。
- 用 `go test -trace` 或服务 trace 分析调度阻塞。

## 代码示例

```bash
go test -bench=. -benchmem ./...
go test -run=^$ -bench=BenchmarkX -cpuprofile cpu.out
go tool pprof cpu.out
```

## 面试追问

追问参考答案：[follow-ups.md](follow-ups.md)

- 如果继续追问“benchmark、pprof 和 trace”的底层机制，应该讲到哪些层次？
- 这个知识点在真实项目里应该如何取舍？
- 最容易踩的坑是什么？为什么这些坑不是背结论就能避免的？
- 如何用测试、工具或 profiling 验证自己的判断？
- 当数据量、并发量或团队规模变大后，这个问题会怎样升级？
