# 029. 有缓冲和无缓冲 channel

## 问题

无缓冲 channel 和有缓冲 channel 有什么区别？

## 先给结论

无缓冲 channel 强调同步交接：发送方和接收方必须同时准备好，通信才完成。

有缓冲 channel 内部有固定容量队列：缓冲未满时发送可以先完成，缓冲非空时接收可以先完成。

缓冲不是越大越好。它只能吸收短暂突发，不能提升长期处理能力。消费者长期处理不过来时，大缓冲只会把问题变成排队、延迟和内存上涨。

## 无缓冲 channel：同步交接

```go
ch := make(chan int)

go func() {
	fmt.Println("before send")
	ch <- 1
	fmt.Println("after send")
}()

time.Sleep(time.Second)
fmt.Println("before receive")
v := <-ch
fmt.Println("received", v)
```

发送方会卡在 `ch <- 1`，直到接收方执行 `<-ch`。

无缓冲 channel 适合：

- 需要严格同步。
- 交接数据所有权。
- 等待某个事件发生。
- 不想让生产者领先消费者。

## 有缓冲 channel：有限排队

```go
ch := make(chan int, 2)

ch <- 1
ch <- 2
// ch <- 3 // 缓冲满，会阻塞

fmt.Println(<-ch)
fmt.Println(<-ch)
```

有缓冲 channel 适合吸收短时间突发：

```go
jobs := make(chan Job, 100)
```

但如果消费者每秒只能处理 10 个任务，生产者每秒来 1000 个任务，容量 10000 也只是延迟暴露问题。

## 缓冲容量就是排队策略

容量应该来自业务约束：

- 最多允许多少任务排队？
- 排队多久仍然有意义？
- 内存能承受多少积压？
- 队列满时是阻塞、丢弃、超时还是返回错误？

示例：队列满时快速失败。

```go
func Submit(ctx context.Context, jobs chan<- Job, job Job) error {
	select {
	case jobs <- job:
		return nil
	case <-ctx.Done():
		return ctx.Err()
	default:
		return ErrQueueFull
	}
}
```

示例：队列满时等待一段时间。

```go
func Submit(ctx context.Context, jobs chan<- Job, job Job) error {
	select {
	case jobs <- job:
		return nil
	case <-ctx.Done():
		return ctx.Err()
	}
}
```

调用方可以给 `ctx` 设置 timeout。

## `len(ch)` 只能观测，不能保证并发正确性

```go
if len(ch) < cap(ch) {
	ch <- job
}
```

这不是安全的非阻塞发送判断。你检查完 `len` 后，其他 goroutine 可能已经把 channel 填满。

正确的非阻塞发送：

```go
select {
case ch <- job:
	return nil
default:
	return ErrQueueFull
}
```

`len(ch)` 可以用于指标观测：

```go
queueDepth.Set(float64(len(jobs)))
```

不要用它做并发控制。

## 关闭时缓冲里的数据还能读

```go
ch := make(chan int, 2)
ch <- 1
ch <- 2
close(ch)

for v := range ch {
	fmt.Println(v)
}
```

关闭 channel 后，已经在缓冲中的值仍然会被接收。读完后，`range` 结束。

如果用双返回值接收：

```go
v, ok := <-ch
if !ok {
	return
}
_ = v
```

`ok=false` 表示 channel 已关闭且没有剩余值。

## 面试追问

追问参考答案：[follow-ups.md](follow-ups.md)

- 无缓冲 channel 的发送和接收什么时候完成？
- 有缓冲 channel 满了或空了会怎样？
- 为什么大缓冲不能解决消费者慢的问题？
- 队列满时有哪些处理策略？
- `len(ch)` 能不能用来判断发送不会阻塞？
- 关闭有缓冲 channel 后，缓冲里的数据还能读吗？
