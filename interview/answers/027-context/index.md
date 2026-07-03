# 027. context

## 问题

`context.Context` 应该怎么用？常见误用有哪些？

## 先给结论

context 用于在调用链上传递取消、超时、截止时间和请求范围值。它不是可选参数容器，也不会自动终止 goroutine，必须由代码主动监听。

## 深入理解

### 1. 这道题真正考察什么

- 是否知道 context 应作为函数第一个参数传递。
- 是否能区分 `WithCancel`、`WithTimeout`、`WithDeadline`。
- 是否知道 cancel 必须调用以释放 timer 和子 context 资源。
- 是否能说明 context value 的使用边界。

### 2. 底层机制要讲清楚

- context 构成树状结构，父 context 取消会传播给子 context。
- `Done()` 返回 channel，关闭表示取消或超时发生。
- `Err()` 可以区分 `Canceled` 和 `DeadlineExceeded`。
- context value 适合请求范围的元数据，不适合传业务参数或可选配置。

### 3. 工程实践怎么取舍

- 入口层创建带超时或取消的 context，向下游传递。
- 启动 goroutine 时把 context 传进去，并在 select 中监听 `ctx.Done()`。
- 使用 `defer cancel()` 释放资源，即使函数正常返回也要调用。
- 日志 trace id、用户认证信息等可以放 value，但要用自定义 key 类型。

### 4. 常见误区

- 把 context 存进结构体长期持有。
- 忘记 cancel，导致 timer 和子 context 资源延迟释放。
- 把业务必填参数塞进 context value，破坏函数签名清晰度。
- 认为 context 取消后正在执行的代码会被强制中断。

## 如何验证理解

- 写测试构造超时 context，确认函数及时返回。
- 用 goroutine 泄漏测试检查取消后后台任务是否退出。
- 在日志中输出 `ctx.Err()`，区分主动取消和超时。

## 代码示例

```go
ctx, cancel := context.WithTimeout(context.Background(), time.Second)
defer cancel()

err := callRemote(ctx)
```

## 面试追问

追问参考答案：[follow-ups.md](follow-ups.md)

- 如果继续追问“context”的底层机制，应该讲到哪些层次？
- 这个知识点在真实项目里应该如何取舍？
- 最容易踩的坑是什么？为什么这些坑不是背结论就能避免的？
- 如何用测试、工具或 profiling 验证自己的判断？
- 当数据量、并发量或团队规模变大后，这个问题会怎样升级？
