# 030. channel 关闭 - 面试追问

## 1. channel 应该由发送方还是接收方关闭？

通常由发送方关闭，更准确地说，由能确定“不会再有发送”的一方关闭。

单生产者：

```go
func produce(out chan<- int) {
	defer close(out)
	for i := 0; i < 3; i++ {
		out <- i
	}
}
```

接收方：

```go
for v := range out {
	fmt.Println(v)
}
```

接收方主动关闭数据 channel，可能导致发送方 panic：

```go
func consume(ch chan int) {
	close(ch) // 不推荐
}
```

## 2. 关闭 channel 后接收会返回什么？

如果缓冲里还有值，先返回缓冲值，`ok=true`。读完后返回零值，`ok=false`。

```go
ch := make(chan int, 1)
ch <- 7
close(ch)

v, ok := <-ch
fmt.Println(v, ok) // 7 true

v, ok = <-ch
fmt.Println(v, ok) // 0 false
```

`for range ch` 会一直读到 `ok=false` 后退出。

## 3. 为什么从关闭 channel 读零值时必须检查 `ok`？

因为零值可能是合法数据。

```go
ch := make(chan int)
close(ch)

v := <-ch
fmt.Println(v) // 0
```

你无法知道这个 `0` 是发送方发送的，还是 channel 关闭后的零值。

正确写法：

```go
v, ok := <-ch
if !ok {
	return
}
use(v)
```

如果用 `range`，Go 会自动处理 `ok`：

```go
for v := range ch {
	use(v)
}
```

## 4. 多个发送方时如何安全关闭 channel？

不要让多个发送方各自 close。用 `WaitGroup` 等所有发送方结束后，由一个协调者关闭。

```go
out := make(chan Result)
var wg sync.WaitGroup

for _, job := range jobs {
	wg.Add(1)
	go func(job Job) {
		defer wg.Done()
		out <- handle(job)
	}(job)
}

go func() {
	wg.Wait()
	close(out)
}()

for r := range out {
	consume(r)
}
```

这样 close 只发生一次，并且一定在所有发送完成之后。

## 5. close 能不能作为取消发送方的机制？

不能关闭数据 channel 来取消发送方。发送方继续发送会 panic。

错误方向：

```go
close(jobs) // 接收方关闭，生产者可能还在 jobs <- job
```

取消应该用 context 或 done channel：

```go
func producer(ctx context.Context, out chan<- Job) {
	for {
		select {
		case out <- nextJob():
		case <-ctx.Done():
			return
		}
	}
}
```

数据 channel 的 close 表示“没有更多数据”；取消信号表示“请停止工作”。两者语义不同。

## 6. channel 是否一定要关闭？

不一定。

只有接收方需要知道结束时才需要关闭，比如：

```go
for v := range ch {
	process(v)
}
```

如果 channel 只是对象内部通信，生命周期随对象结束，没有 goroutine 等待结束信号，可以不关闭。

关闭不是释放内存的必要动作。channel 没有引用后会被 GC 回收。

面试里可以总结：需要通知接收方“不会再有值”才 close；否则不必为了形式主义 close。
