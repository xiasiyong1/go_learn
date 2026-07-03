# 033. WaitGroup

## 问题

`WaitGroup` 怎么用？常见错误有哪些？

## 先给结论

WaitGroup 用于等待一组 goroutine 完成。它只负责计数等待，不负责错误收集、取消传播或并发限制，这些能力需要额外设计。

## 深入理解

### 1. 这道题真正考察什么

- 是否知道 Add、Done、Wait 的计数关系。
- 是否知道 Add 应该在启动 goroutine 前调用。
- 是否能解释计数变成负数会 panic。
- 是否知道 WaitGroup 可以复用，但必须在上一轮 Wait 返回后再开始新一轮。

### 2. 底层机制要讲清楚

- WaitGroup 内部维护计数和等待者数量。
- `Done()` 等价于 `Add(-1)`。
- 如果 goroutine 已经开始甚至结束后才 Add，Wait 可能提前返回。
- WaitGroup 不传递 goroutine 的返回值，因此错误需要 channel、mutex 聚合或 errgroup。

### 3. 工程实践怎么取舍

- 启动 goroutine 前调用 `wg.Add(1)`，goroutine 内 `defer wg.Done()`。
- 需要错误和取消时优先考虑 `errgroup.WithContext`。
- 需要限制并发时配合 semaphore、worker pool 或 errgroup 的限制能力。
- 不要把 WaitGroup 复制传递，通常传指针或作为结构体字段谨慎使用。

### 4. 常见误区

- 在 goroutine 内部 Add，导致 Wait 和 Add 竞态。
- 某个分支忘记 Done，Wait 永久阻塞。
- Done 调多了，计数为负 panic。
- 用 WaitGroup 等待任务，却没有处理任务失败后的取消。

## 如何验证理解

- 为并发函数写测试，确保所有分支都会 Done。
- 使用 race detector 检查 Add/Wait 是否存在竞态。
- 故意制造错误任务，验证错误聚合和取消逻辑。

## 代码示例

```go
var wg sync.WaitGroup
for _, job := range jobs {
	wg.Add(1)
	go func(job Job) {
		defer wg.Done()
		process(job)
	}(job)
}
wg.Wait()
```

## 面试追问

追问参考答案：[follow-ups.md](follow-ups.md)

- 如果继续追问“WaitGroup”的底层机制，应该讲到哪些层次？
- 这个知识点在真实项目里应该如何取舍？
- 最容易踩的坑是什么？为什么这些坑不是背结论就能避免的？
- 如何用测试、工具或 profiling 验证自己的判断？
- 当数据量、并发量或团队规模变大后，这个问题会怎样升级？
