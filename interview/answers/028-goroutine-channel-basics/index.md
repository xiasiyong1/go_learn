# 028. goroutine 和 channel 基础

## 问题

goroutine 和 channel 的基本通信模型是什么？

## 先给结论

goroutine 是 Go 里的并发执行单元，创建成本比 OS 线程低，由 Go runtime 调度到 OS 线程上运行。

channel 是类型安全的通信和同步原语。它可以传数据，也可以表达同步、所有权转移和退出信号。

但 channel 不是“比锁高级”的替代品。共享状态可以用锁保护，数据流水线和所有权交接更适合 channel。关键是选择更清楚的并发模型。

## 启动 goroutine

```go
go func() {
	fmt.Println("hello")
}()
```

启动 goroutine 后，调用方不会等待它完成。下面这段程序可能什么都来不及输出：

```go
func main() {
	go fmt.Println("hello")
}
```

如果要等待，使用 `sync.WaitGroup`：

```go
var wg sync.WaitGroup
wg.Add(1)

go func() {
	defer wg.Done()
	fmt.Println("hello")
}()

wg.Wait()
```

面试里要强调：启动 goroutine 时，要同时设计退出、等待、错误处理。

## channel 基本通信

```go
ch := make(chan int)

go func() {
	ch <- 1
}()

v := <-ch
fmt.Println(v)
```

无缓冲 channel 上，发送和接收要同时准备好，通信才完成。这既是传值，也是同步。

有缓冲 channel：

```go
ch := make(chan int, 2)
ch <- 1
ch <- 2

fmt.Println(<-ch)
fmt.Println(<-ch)
```

缓冲未满时发送可以先完成；缓冲为空时接收会阻塞。

## channel 适合所有权交接

```go
func producer(out chan<- Job) {
	defer close(out)
	for i := 0; i < 10; i++ {
		out <- Job{ID: i}
	}
}

func consumer(in <-chan Job) {
	for job := range in {
		handle(job)
	}
}
```

这里 `producer` 生成 job，通过 channel 交给 `consumer`。所有权流向很清楚。

方向类型能让 API 更明确：

```go
func producer(out chan<- Job) {}
func consumer(in <-chan Job) {}
```

`chan<- Job` 只能发送，`<-chan Job` 只能接收。

## 什么时候用锁更简单

如果多个 goroutine 频繁读写同一个内存结构，锁通常更直接。

```go
type Counter struct {
	mu sync.Mutex
	n  int
}

func (c *Counter) Inc() {
	c.mu.Lock()
	defer c.mu.Unlock()
	c.n++
}
```

不要为了“使用 channel”把简单状态保护写成复杂消息循环。

channel 更适合：

- pipeline。
- worker pool。
- fan-in / fan-out。
- 任务队列。
- 所有权交接。
- 取消信号。

mutex 更适合：

- 保护共享 map。
- 保护计数器或状态结构。
- 多个操作需要维护同一个不变量。

## goroutine 泄漏的基本例子

```go
func leak() {
	ch := make(chan int)
	go func() {
		ch <- 1 // 没有人接收，永久阻塞
	}()
}
```

修正方向：让发送可以取消，或者保证接收。

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

只要创建 goroutine，就要知道它在什么条件下退出。

## 错误传播不能丢

不推荐：

```go
go func() {
	if err := doWork(); err != nil {
		log.Println(err)
	}
}()
```

调用方不知道任务是否失败。

简单场景可以用 error channel：

```go
errCh := make(chan error, 1)

go func() {
	errCh <- doWork()
}()

if err := <-errCh; err != nil {
	return err
}
```

多个 goroutine 的错误收集可以用 `errgroup`，后面并发错误处理题会继续展开。

## 面试追问

追问参考答案：[follow-ups.md](follow-ups.md)

- goroutine 启动后主函数会自动等待吗？
- channel 通信为什么也能表达同步？
- 什么时候应该用 channel，什么时候用 mutex 更清楚？
- channel 方向类型有什么意义？
- goroutine 泄漏通常卡在哪些阻塞点？
- 并发任务的错误应该怎么返回给调用方？
