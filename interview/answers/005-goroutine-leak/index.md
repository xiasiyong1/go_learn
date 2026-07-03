# 005. goroutine 泄漏

## 问题

goroutine 泄漏通常是怎么产生的？如何排查和避免？

## 先给结论

goroutine 泄漏本质上是 goroutine 永远无法走到退出路径。常见原因是 channel 阻塞、上下文未取消、后台循环无停止条件、I/O 无超时或生产消费关系断裂。

## 深入理解

### 1. 这道题真正考察什么

- 是否能把泄漏归因到具体阻塞点，而不是只说“goroutine 太多”。
- 是否知道 goroutine 泄漏会连带持有栈、堆对象、连接、锁和上下文。
- 是否能设计取消、超时、关闭 channel 和错误传播机制。
- 是否会用 pprof、trace、日志和测试观察 goroutine 数量趋势。

### 2. 底层机制要讲清楚

- goroutine 初始栈很小，但不是零成本；阻塞 goroutine 仍可能通过栈引用持有大量堆对象。
- channel 发送和接收都需要配对；无人接收的发送和无人发送的接收都会阻塞。
- `context.Context` 是取消信号传播，不会自动杀死 goroutine；goroutine 必须主动 select `ctx.Done()`。
- 泄漏通常不是瞬间崩溃，而是随着请求量逐渐积累，最终表现为内存、fd、连接池或调度压力异常。

### 3. 工程实践怎么取舍

- 启动 goroutine 时同步设计退出条件，谁创建谁负责管理生命周期。
- 所有可能阻塞的外部调用都设置 timeout、deadline 或 context。
- pipeline 中下游提前退出时，要通知上游停止生产。
- 服务关闭时等待后台任务退出，并给退出过程设置上限时间。

### 4. 常见误区

- 只在外层请求有 context，内部 goroutine 不监听。
- 只关闭数据 channel，不处理错误 channel 或 done channel。
- 用 goroutine 包一层调用来“防止阻塞”，却不回收结果。
- 排查时只看 goroutine 数量，不看堆栈停在哪一行。

## 如何验证理解

- 用 `runtime.NumGoroutine()` 做趋势观察，不把单个瞬时值当结论。
- 用 `go tool pprof http://host/debug/pprof/goroutine` 查看 goroutine 堆栈分组。
- 压测后停止流量，观察 goroutine 数量是否回落到基线。

## 代码示例

```go
package main

import (
	"context"
	"fmt"
	"time"
)

func worker(ctx context.Context, jobs <-chan int) {
	for {
		select {
		case <-ctx.Done():
			return
		case job, ok := <-jobs:
			if !ok {
				return
			}
			fmt.Println(job)
		}
	}
}

func main() {
	ctx, cancel := context.WithTimeout(context.Background(), time.Second)
	defer cancel()

	jobs := make(chan int)
	go worker(ctx, jobs)

	jobs <- 1
	close(jobs)
}
```

## 面试追问

追问参考答案：[follow-ups.md](follow-ups.md)

- 如果继续追问“goroutine 泄漏”的底层机制，应该讲到哪些层次？
- 这个知识点在真实项目里应该如何取舍？
- 最容易踩的坑是什么？为什么这些坑不是背结论就能避免的？
- 如何用测试、工具或 profiling 验证自己的判断？
- 当数据量、并发量或团队规模变大后，这个问题会怎样升级？
