# 028. goroutine 和 channel 基础 - 面试追问

## 1. goroutine 启动后主函数会自动等待吗？

不会。

```go
func main() {
	go fmt.Println("hello")
}
```

`main` 返回后进程结束，goroutine 可能还没执行。

需要等待时，用 `sync.WaitGroup`：

```go
var wg sync.WaitGroup
wg.Add(1)

go func() {
	defer wg.Done()
	fmt.Println("hello")
}()

wg.Wait()
```

如果需要拿结果，用 channel：

```go
resultCh := make(chan int, 1)
go func() {
	resultCh <- compute()
}()

result := <-resultCh
```

## 2. channel 通信为什么也能表达同步？

无缓冲 channel 的发送和接收必须同时准备好。

```go
ch := make(chan struct{})

go func() {
	doWork()
	close(ch)
}()

<-ch // 等待 doWork 完成
```

这里 channel 没传业务数据，只用于同步完成信号。

传数据时也有同步含义：

```go
ch := make(chan int)
go func() {
	x := 1
	ch <- x
}()

fmt.Println(<-ch)
```

发送完成前的写入，对接收方是可见的。

## 3. 什么时候应该用 channel，什么时候用 mutex 更清楚？

channel 适合数据流和所有权交接：

```go
jobs := make(chan Job)
go producer(jobs)
consumer(jobs)
```

mutex 适合保护共享状态：

```go
type Cache struct {
	mu sync.RWMutex
	m  map[string]User
}
```

如果你发现自己用 channel 传各种命令，只是为了保护一个 map，可以考虑普通 map 加锁是否更直接。

经验是：

- 数据从 A 流到 B：channel。
- 多个 goroutine 共同读写一个状态：mutex。
- 单一 goroutine 拥有状态、其他 goroutine 发请求：channel owner 模型也可以。

## 4. channel 方向类型有什么意义？

它能把函数能力限制得更清楚。

```go
func produce(out chan<- int) {
	out <- 1
	close(out)
}

func consume(in <-chan int) {
	for v := range in {
		fmt.Println(v)
	}
}
```

`chan<- int` 只能发送，不能接收。`<-chan int` 只能接收，不能发送。

这让 API 表达出职责：谁生产、谁消费、谁有资格关闭。

## 5. goroutine 泄漏通常卡在哪些阻塞点？

常见阻塞点：

- 发送没人接收。
- 接收没人发送，也没人关闭。
- select 没有监听取消。
- I/O 没有 timeout。
- 等锁或 WaitGroup 永远等不到。

发送阻塞例子：

```go
func leak() {
	ch := make(chan int)
	go func() {
		ch <- 1
	}()
}
```

修正：

```go
func send(ctx context.Context, ch chan<- int) {
	go func() {
		select {
		case ch <- 1:
		case <-ctx.Done():
			return
		}
	}()
}
```

## 6. 并发任务的错误应该怎么返回给调用方？

单个 goroutine 可以用带缓冲 error channel：

```go
errCh := make(chan error, 1)

go func() {
	errCh <- doWork()
}()

if err := <-errCh; err != nil {
	return err
}
```

带缓冲是为了避免调用方超时返回后，发送错误的 goroutine 卡住。

多个 goroutine 可以收集多个错误，或使用 `errgroup` 这类工具处理“第一个错误取消其他任务”的模式。

无论哪种方式，都不要只在 goroutine 里打印错误然后丢掉，调用方需要知道任务是否成功。
