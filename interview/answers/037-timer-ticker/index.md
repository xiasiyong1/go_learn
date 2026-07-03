# 037. Timer 和 Ticker

## 问题

`time.Timer` 和 `time.Ticker` 有哪些资源释放问题？

## 先给结论

Timer 表示一次性时间事件，Ticker 表示周期性事件。它们的难点在 Stop、Reset、资源释放和循环中创建 timer 的成本。

## 深入理解

### 1. 这道题真正考察什么

- 是否知道 Timer 只触发一次，Ticker 持续触发直到 Stop。
- 是否理解 Stop 的返回值和 channel 中可能已有值的关系。
- 是否知道 Reset 前要处理旧 timer 状态。
- 是否能解释循环中 `time.After` 的资源和分配问题。

### 2. 底层机制要讲清楚

- Timer 到期后会尝试向自己的 channel 发送时间值。
- Stop 返回 false 表示 timer 已到期或已停止，channel 里可能还有未读值。
- Reset 复用 timer，但必须避免旧事件和新事件混在一起。
- Ticker 如果不 Stop，会持续关联运行时计时器资源。

### 3. 工程实践怎么取舍

- 单次简单超时可用 `time.After`。
- 循环或高频超时逻辑使用 `time.NewTimer` 并复用。
- 周期任务用 Ticker，并在退出路径 `defer ticker.Stop()`。
- 重置 timer 前按照官方推荐处理 Stop 和 drain，避免读到旧事件。

### 4. 常见误区

- 循环里每次 select 都创建 `time.After`，造成大量 timer。
- Ticker 不 Stop，后台资源持续存在。
- Stop 返回 false 后没有 drain，下一轮读到旧超时。
- 以为 Ticker 会补齐所有错过的 tick，实际慢消费者会丢或合并节奏。

## 如何验证理解

- 写单测覆盖 timer 被提前 Stop、到期后 Reset 的边界。
- 用 benchmark 看 `time.After` 循环的分配。
- 用 goroutine 和 heap profile 排查 timer/ticker 使用不当造成的资源增长。

## 代码示例

```go
ticker := time.NewTicker(time.Second)
defer ticker.Stop()

for {
	select {
	case <-ticker.C:
		doWork()
	case <-ctx.Done():
		return
	}
}
```

## 面试追问

追问参考答案：[follow-ups.md](follow-ups.md)

- 如果继续追问“Timer 和 Ticker”的底层机制，应该讲到哪些层次？
- 这个知识点在真实项目里应该如何取舍？
- 最容易踩的坑是什么？为什么这些坑不是背结论就能避免的？
- 如何用测试、工具或 profiling 验证自己的判断？
- 当数据量、并发量或团队规模变大后，这个问题会怎样升级？
