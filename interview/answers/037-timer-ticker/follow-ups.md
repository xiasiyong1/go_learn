# 037. Timer 和 Ticker - 面试追问

## 1. `Timer` 和 `Ticker` 的语义区别是什么？

`Timer` 触发一次：

```go
timer := time.NewTimer(time.Second)
defer timer.Stop()

<-timer.C
fmt.Println("timeout")
```

`Ticker` 周期触发：

```go
ticker := time.NewTicker(time.Second)
defer ticker.Stop()

for t := range ticker.C {
	fmt.Println(t)
}
```

Timer 适合单次 deadline；Ticker 适合心跳、周期上报、定期清理。

## 2. 为什么循环里反复 `time.After` 要谨慎？

`time.After` 每次调用都会创建 timer。

```go
for {
	select {
	case <-time.After(time.Millisecond):
		tick()
	case <-ctx.Done():
		return
	}
}
```

高频循环里会带来额外分配和 timer 管理成本。

周期任务用 ticker：

```go
ticker := time.NewTicker(time.Millisecond)
defer ticker.Stop()
```

每轮独立超时且频率很高时，可以封装复用 Timer。

## 3. `Ticker` 为什么要 `Stop()`？

因为 ticker 表示一个持续的周期事件。你不再需要它时，应该停止。

```go
func Run(ctx context.Context) {
	ticker := time.NewTicker(time.Second)
	defer ticker.Stop()

	for {
		select {
		case <-ticker.C:
			report()
		case <-ctx.Done():
			return
		}
	}
}
```

`Stop()` 同时让生命周期更清楚：函数退出后这个周期任务不再存在。

## 4. `Ticker` 会不会补齐慢消费者错过的 tick？

不会按任务队列语义补齐。

```go
ticker := time.NewTicker(100 * time.Millisecond)
defer ticker.Stop()

for range ticker.C {
	time.Sleep(time.Second)
	fmt.Println("tick")
}
```

消费者慢时，不应该期待积压 10 个 tick 然后逐个处理。

如果每个周期任务都必须执行，要设计可靠队列或根据当前时间计算补偿，而不是依赖 ticker channel 保存所有事件。

## 5. `Timer.Stop()` 返回 false 说明什么？

说明 timer 已经停止或已经触发。它不等于“channel 一定有值”，但常见场景下需要考虑 channel 里可能有旧值。

```go
if !timer.Stop() {
	select {
	case <-timer.C:
	default:
	}
}
```

这个非阻塞 drain 可以避免读到旧超时事件。

注意：timer 代码很容易受并发读写影响。最好让一个 goroutine 拥有 timer，避免多个 goroutine 同时 Stop/Reset/读 C。

## 6. `Timer.Reset()` 前为什么要处理旧状态？

因为旧 timer 可能已经触发，channel 里可能还有旧事件。如果直接 Reset，下一轮 select 可能立刻读到旧事件。

封装：

```go
func resetTimer(t *time.Timer, d time.Duration) {
	if !t.Stop() {
		select {
		case <-t.C:
		default:
		}
	}
	t.Reset(d)
}
```

这类代码适合封装在底层工具里。普通业务代码如果只是单次超时，用 `context.WithTimeout` 或 `time.After` 更清楚。
