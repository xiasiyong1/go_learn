# 048. goroutine 栈

## 问题

goroutine 栈如何增长？为什么 goroutine 可以很轻量？

## 核心答案

goroutine 初始栈很小，运行时会按需增长和收缩。相比系统线程固定较大栈空间，goroutine 可以用更低成本创建大量并发任务。

栈增长由 runtime 管理，函数调用前会检查栈空间是否足够，不够时触发扩栈。

## 代码示例

```go
go func() {
	doWork()
}()
```

## 面试追问

- goroutine 数量是否可以无限增长？
- 深递归对 goroutine 栈有什么影响？
