# 030. channel 关闭

## 问题

channel 应该由谁关闭？向已关闭 channel 发送会怎样？

## 先给结论

关闭 channel 表示“不会再发送新值”。通常应该由发送方关闭，准确地说，是由“能确定没有任何发送者会再发送”的一方关闭。

基本规则：

- 向已关闭 channel 发送会 panic。
- 关闭已关闭 channel 会 panic。
- 关闭 nil channel 会 panic。
- 从已关闭 channel 接收会立即返回零值和 `ok=false`，但如果缓冲里还有值，会先读完缓冲值。
- channel 不一定必须关闭；只有接收方需要知道“没有更多数据”时才需要关闭。

## 单生产者关闭数据 channel

```go
func producer(out chan<- int) {
	defer close(out)

	for i := 0; i < 3; i++ {
		out <- i
	}
}

func consumer(in <-chan int) {
	for v := range in {
		fmt.Println(v)
	}
}
```

生产者知道自己不会再发送，所以由生产者关闭。

接收方不要关闭数据 channel：

```go
func badConsumer(in chan int) {
	close(in) // 可能导致发送方 panic
}
```

## 向已关闭 channel 发送会 panic

```go
ch := make(chan int)
close(ch)

ch <- 1 // panic: send on closed channel
```

关闭已关闭 channel 也会 panic：

```go
close(ch) // panic: close of closed channel
```

不要用 recover 掩盖这个问题。它通常说明关闭责任设计错了。

## 从已关闭 channel 接收

```go
ch := make(chan int, 1)
ch <- 42
close(ch)

v, ok := <-ch
fmt.Println(v, ok) // 42 true

v, ok = <-ch
fmt.Println(v, ok) // 0 false
```

如果元素类型的零值也是合法数据，必须检查 `ok`。

```go
v := <-ch // 不推荐：无法区分收到 0 还是 channel 已关闭
_ = v
```

推荐：

```go
v, ok := <-ch
if !ok {
	return
}
_ = v
```

## 多生产者如何关闭

多个生产者不能各自 close 同一个 channel：

```go
for i := 0; i < 3; i++ {
	go func() {
		defer close(out) // 错误：多个 goroutine 竞争 close
		out <- work()
	}()
}
```

正确做法是由协调者等待所有生产者结束后关闭：

```go
out := make(chan Result)
var wg sync.WaitGroup

for i := 0; i < 3; i++ {
	wg.Add(1)
	go func(id int) {
		defer wg.Done()
		out <- work(id)
	}(i)
}

go func() {
	wg.Wait()
	close(out)
}()

for r := range out {
	handle(r)
}
```

关闭权集中在一个 goroutine 里，逻辑最清楚。

## 关闭不是取消发送方的万能手段

关闭数据 channel 是告诉接收方没有更多数据，不是告诉发送方停止。

如果要通知多个 goroutine 停止，可以使用 `context` 或单独的 done channel：

```go
func worker(ctx context.Context, jobs <-chan Job) {
	for {
		select {
		case <-ctx.Done():
			return
		case job, ok := <-jobs:
			if !ok {
				return
			}
			handle(job)
		}
	}
}
```

不要让接收方通过关闭 `jobs` 来通知生产者停止，那会让生产者发送时 panic。

## channel 不一定要关闭

如果没有接收方在等待“结束信号”，可以不关闭 channel。

```go
type Notifier struct {
	events chan Event
}
```

对象整体不再使用后，channel 会随对象一起被 GC 回收。关闭 channel 不是释放内存的必要步骤。

需要关闭的典型场景是：

- 接收方用 `for range ch` 等待结束。
- 需要广播“没有更多数据”。
- 需要让阻塞接收者退出。

## 面试追问

追问参考答案：[follow-ups.md](follow-ups.md)

- channel 应该由发送方还是接收方关闭？
- 关闭 channel 后接收会返回什么？
- 为什么从关闭 channel 读零值时必须检查 `ok`？
- 多个发送方时如何安全关闭 channel？
- close 能不能作为取消发送方的机制？
- channel 是否一定要关闭？
