# 029. 有缓冲和无缓冲 channel - 面试追问

## 1. 无缓冲 channel 的发送和接收什么时候完成？

发送方和接收方都准备好时才完成。

```go
ch := make(chan int)

go func() {
	ch <- 1 // 等待接收方
}()

v := <-ch // 等待发送方
fmt.Println(v)
```

无缓冲 channel 的一次通信同时完成数据传递和同步。

如果没有接收方，发送会阻塞：

```go
ch := make(chan int)
ch <- 1 // deadlock
```

## 2. 有缓冲 channel 满了或空了会怎样？

缓冲未满，发送可以完成；缓冲满了，发送阻塞。

```go
ch := make(chan int, 1)
ch <- 1
// ch <- 2 // 阻塞
```

缓冲非空，接收可以完成；缓冲空了，接收阻塞。

```go
fmt.Println(<-ch)
// fmt.Println(<-ch) // 阻塞
```

如果希望阻塞可取消，用 select：

```go
select {
case ch <- job:
	return nil
case <-ctx.Done():
	return ctx.Err()
}
```

## 3. 为什么大缓冲不能解决消费者慢的问题？

因为长期吞吐取决于消费者处理速度。

```go
jobs := make(chan Job, 10000)
```

如果生产速度是每秒 1000，消费速度是每秒 100，队列迟早会满。大缓冲只是在短期内让生产者不阻塞，代价是：

- 更多内存占用。
- 更长排队时间。
- 更高尾延迟。
- 故障更晚暴露。

真正解决办法是提高消费能力、限流、丢弃、降级或返回错误。

## 4. 队列满时有哪些处理策略？

阻塞等待：

```go
jobs <- job
```

支持取消：

```go
select {
case jobs <- job:
	return nil
case <-ctx.Done():
	return ctx.Err()
}
```

快速失败：

```go
select {
case jobs <- job:
	return nil
default:
	return ErrQueueFull
}
```

丢弃旧数据或新数据也可以，但要明确业务语义。比如监控采样可以丢，支付任务通常不能丢。

## 5. `len(ch)` 能不能用来判断发送不会阻塞？

不能。

```go
if len(ch) < cap(ch) {
	ch <- job
}
```

在并发环境中，检查和发送之间可能被其他 goroutine 插入，导致 channel 已满。

正确非阻塞发送：

```go
select {
case ch <- job:
	return true
default:
	return false
}
```

`len(ch)` 可以做指标：

```go
queueDepth.Set(float64(len(ch)))
```

但不要用它保证并发正确性。

## 6. 关闭有缓冲 channel 后，缓冲里的数据还能读吗？

能。

```go
ch := make(chan int, 2)
ch <- 1
ch <- 2
close(ch)

fmt.Println(<-ch) // 1
fmt.Println(<-ch) // 2

v, ok := <-ch
fmt.Println(v, ok) // 0 false
```

`range` 会把缓冲里的值读完，然后退出：

```go
for v := range ch {
	fmt.Println(v)
}
```

关闭表示“不会再有新值”，不是“丢弃缓冲里的旧值”。
