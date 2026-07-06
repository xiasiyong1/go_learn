# 037. Timer 和 Ticker

## 问题

`time.Timer` 和 `time.Ticker` 有哪些资源释放问题？

## 先给结论

`Timer` 是一次性定时事件，`Ticker` 是周期性定时事件。

常见规则：

- 单次简单超时可以用 `time.After`。
- 循环或高频超时逻辑不要反复 `time.After`，优先复用 `Timer` 或使用 `Ticker`。
- `Ticker` 用完要 `Stop()`。
- `Timer.Reset` 前要清楚旧 timer 是否已经触发，避免读到旧事件。
- 慢消费者不会收到每一个 tick，不能把 ticker 当精确任务队列。

## 单次超时：time.After

```go
select {
case result := <-resultCh:
	return result, nil
case <-time.After(time.Second):
	return Result{}, fmt.Errorf("timeout")
}
```

这适合单次等待。代码简单，可读性好。

如果在循环里每次都调用 `time.After`：

```go
for {
	select {
	case item := <-items:
		handle(item)
	case <-time.After(time.Second):
		report()
	}
}
```

每轮都会创建新的 timer，高频路径会有额外成本。

## 周期任务：Ticker

```go
ticker := time.NewTicker(time.Second)
defer ticker.Stop()

for {
	select {
	case <-ticker.C:
		report()
	case <-ctx.Done():
		return ctx.Err()
	}
}
```

`Stop()` 很重要：它表达这个周期事件的生命周期结束了。长期服务里不要让后台 ticker 没有退出路径。

## Ticker 不保证补齐所有 tick

```go
ticker := time.NewTicker(100 * time.Millisecond)
defer ticker.Stop()

for range ticker.C {
	time.Sleep(time.Second)
}
```

消费者处理很慢时，ticker 不会无限堆积 tick 让你补齐。它适合“按节奏触发”，不适合表达“每一个周期任务都必须执行一次”的可靠队列。

如果每个周期任务都不能丢，要显式设计任务队列、持久化或补偿机制。

## Timer 和 Stop

```go
timer := time.NewTimer(time.Second)
defer timer.Stop()

select {
case <-timer.C:
	return fmt.Errorf("timeout")
case result := <-resultCh:
	if !timer.Stop() {
		<-timer.C
	}
	return result.Err
}
```

这里有一个细节：`Stop` 返回 false 可能表示 timer 已经触发，channel 里可能还有值。为了避免后续复用 timer 时读到旧值，常见做法是 drain。

不过实际写法要避免在 channel 已经被其他分支读走时盲目 drain，否则可能阻塞。复杂 timer 复用最好封装起来，别在业务代码里散落。

## Timer Reset

复用 timer 时，核心是避免旧事件和新事件混在一起。

简单场景可以这样封装：

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

使用：

```go
timer := time.NewTimer(time.Second)
defer timer.Stop()

for {
	resetTimer(timer, time.Second)
	select {
	case <-timer.C:
		return fmt.Errorf("idle timeout")
	case msg := <-messages:
		handle(msg)
	case <-ctx.Done():
		return ctx.Err()
	}
}
```

Timer 的 Stop/Reset 细节比较容易写错。低频代码用简单方式；高频路径再考虑复用和封装。

## 面试追问

追问参考答案：[follow-ups.md](follow-ups.md)

- `Timer` 和 `Ticker` 的语义区别是什么？
- 为什么循环里反复 `time.After` 要谨慎？
- `Ticker` 为什么要 `Stop()`？
- `Ticker` 会不会补齐慢消费者错过的 tick？
- `Timer.Stop()` 返回 false 说明什么？
- `Timer.Reset()` 前为什么要处理旧状态？
