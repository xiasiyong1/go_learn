# 050. benchmark、pprof 和 trace

## 问题

如何用 benchmark、pprof 和 trace 定位性能问题？

## 核心答案

benchmark 用来稳定复现某段代码的性能，重点看耗时、分配次数和分配字节数。

pprof 用于分析 CPU、内存、goroutine、锁竞争等画像，找出主要成本来源。

trace 可以观察 goroutine 调度、网络阻塞、系统调用、GC 等时间线，适合分析并发行为。

## 代码示例

```bash
go test -bench=. -benchmem ./...
go test -run=^$ -bench=BenchmarkX -cpuprofile cpu.out
go tool pprof cpu.out
```

## 面试追问

追问参考答案：[follow-ups.md](follow-ups.md)

- flat 和 cum 分别表示什么？
- 为什么优化前要先测量？
