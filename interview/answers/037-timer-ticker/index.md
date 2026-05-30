# 037. Timer 和 Ticker

## 问题

`time.Timer` 和 `time.Ticker` 有哪些资源释放问题？

## 核心答案

`Timer` 表示一次性定时，`Ticker` 表示周期性触发。使用完后应该调用 `Stop`，尤其是 `Ticker`，否则可能导致后台资源持续存在。

在循环里频繁使用 `time.After` 会创建大量临时定时器，热点路径中建议复用 `Timer`。

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

- `Timer.Stop` 返回值表示什么？
- 重置 Timer 前需要注意什么？
