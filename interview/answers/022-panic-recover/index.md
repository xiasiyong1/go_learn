# 022. panic 和 recover

## 问题

`panic` 和 `recover` 的使用边界是什么？

## 先给结论

panic 表示当前调用栈无法继续的异常状态，recover 只能在同一 goroutine 的 defer 中截获 panic。它们不应该替代普通 error，而应保留给不可恢复或边界兜底场景。

## 深入理解

### 1. 这道题真正考察什么

- 是否知道 panic 会触发当前 goroutine 的 defer 链。
- 是否知道 recover 只有在 defer 函数中直接调用才有效。
- 是否知道 recover 不能捕获其他 goroutine 的 panic。
- 是否能区分程序员错误、不可恢复错误和普通业务错误。

### 2. 底层机制要讲清楚

- panic 发生后，函数停止正常执行，开始沿调用栈执行 defer。
- recover 成功后，当前 goroutine 可以从 defer 返回，函数返回其返回值。
- 不同 goroutine 有独立调用栈，一个 goroutine 的 recover 不能跨栈捕获另一个 goroutine 的 panic。
- runtime 的 fatal error 通常不可 recover，例如并发 map 写导致的 fatal error。

### 3. 工程实践怎么取舍

- 库函数通常返回 error，不要让调用方被迫 recover。
- HTTP middleware、任务执行器、goroutine 启动边界可以 recover 并记录堆栈，避免进程直接崩溃。
- recover 后要把 panic 转成明确错误或日志，不要静默吞掉。
- 内部不变量被破坏时可以 panic，但要让边界清晰。

### 4. 常见误区

- 把 panic 当异常机制写业务分支。
- 在 goroutine 外层没有 recover，导致后台任务 panic 杀死进程。
- recover 后不记录堆栈，问题根因丢失。
- 以为所有 panic 都能 recover，包括 fatal error。

## 如何验证理解

- 写小例子验证 recover 的调用位置和 goroutine 边界。
- 为 goroutine wrapper 写单测，确认 panic 会被转成错误或日志。
- 在线上服务中配合 structured log 输出 panic 值和 stack trace。

## 代码示例

```go
defer func() {
	if r := recover(); r != nil {
		log.Println("panic:", r)
	}
}()
```

## 面试追问

追问参考答案：[follow-ups.md](follow-ups.md)

- 如果继续追问“panic 和 recover”的底层机制，应该讲到哪些层次？
- 这个知识点在真实项目里应该如何取舍？
- 最容易踩的坑是什么？为什么这些坑不是背结论就能避免的？
- 如何用测试、工具或 profiling 验证自己的判断？
- 当数据量、并发量或团队规模变大后，这个问题会怎样升级？
