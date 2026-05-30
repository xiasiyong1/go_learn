# 005. goroutine 泄漏

## 问题

goroutine 泄漏通常是怎么产生的？如何排查和避免？

## 答案

goroutine 泄漏是指 goroutine 因为阻塞、等待或无法退出而长期存活，导致内存、调度和外部资源被持续占用。

常见原因：

- channel 发送或接收没人配对。
- 后台 goroutine 没有退出条件。
- 没有处理超时、取消或上游退出。
- 生产者退出后没有关闭 channel，消费者一直等待。

常见避免方式：

- 使用 `context.Context` 传递取消信号。
- 明确 channel 的关闭方。
- 为 I/O、RPC、锁等待设置超时。
- 在测试或压测中观察 goroutine 数量是否持续增长。

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

- `context` 应该由谁创建，谁取消？
- 如何用 `pprof` 排查 goroutine 泄漏？
