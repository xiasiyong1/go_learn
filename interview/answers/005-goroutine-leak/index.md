# 005. goroutine 泄漏

## 问题

goroutine 泄漏通常是怎么产生的？如何排查和避免？

## 先给结论

goroutine 泄漏的本质是：启动出去的 goroutine 永远走不到退出路径。

常见原因包括：

- channel 发送没人接收。
- channel 接收没人发送，也没人关闭。
- 后台循环没有停止条件。
- `context` 创建了，但 goroutine 没有监听 `ctx.Done()`。
- I/O、RPC、数据库调用没有超时。
- pipeline 下游提前退出，上游还在生产。
- ticker、timer、连接、文件等资源没有释放。

面试时不要只说“用 pprof 查”。更好的回答是：先说明泄漏卡在什么阻塞点，再说明如何设计退出信号，最后说明如何用 pprof、trace、测试和压测验证。

## 泄漏不是 goroutine 数量多，而是不能退出

goroutine 很轻量，但不是零成本。一个泄漏的 goroutine 可能持有：

- goroutine 栈。
- 栈上引用到的堆对象。
- channel、锁、连接、文件描述符。
- request context、用户数据、缓存对象。

所以泄漏经常不是立刻崩溃，而是请求量上来后，内存、连接数、文件描述符或调度开销逐渐升高。

## 场景一：发送没人接收

```go
func sendLater(ch chan<- int) {
	go func() {
		ch <- 1 // 如果没人接收，会一直阻塞
	}()
}
```

如果调用方提前返回，不再接收 `ch`，这个 goroutine 就会卡住。

一种修正方式是让发送方也监听取消信号：

```go
func sendLater(ctx context.Context, ch chan<- int) {
	go func() {
		select {
		case ch <- 1:
		case <-ctx.Done():
			return
		}
	}()
}
```

这里的核心不是“加了 context 就安全”，而是 goroutine 的阻塞点真正 select 了 `ctx.Done()`。

## 场景二：接收没人发送

```go
func waitResult(ch <-chan int) {
	go func() {
		result := <-ch // 如果没人发送，也没人关闭，会一直阻塞
		fmt.Println(result)
	}()
}
```

修正方式之一是给退出路径：

```go
func waitResult(ctx context.Context, ch <-chan int) {
	go func() {
		select {
		case result, ok := <-ch:
			if !ok {
				return
			}
			fmt.Println(result)
		case <-ctx.Done():
			return
		}
	}()
}
```

如果 channel 表示数据流，生产方结束时应该关闭 channel，让消费者能退出：

```go
func producer(out chan<- int) {
	defer close(out)

	for i := 0; i < 3; i++ {
		out <- i
	}
}
```

## 场景三：后台循环没有停止条件

错误示例：

```go
func startReporter() {
	go func() {
		for {
			time.Sleep(time.Second)
			report()
		}
	}()
}
```

这个 goroutine 没有任何退出方式。服务关闭、测试结束、调用方不再需要它时，它仍然活着。

更好的写法：

```go
func startReporter(ctx context.Context) {
	go func() {
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
	}()
}
```

这里同时解决了两个问题：

- goroutine 能响应取消。
- ticker 能释放底层资源。

## 场景四：pipeline 下游提前退出

这是 Go 并发题里很常见的追问。

错误示例：

```go
func gen() <-chan int {
	out := make(chan int)
	go func() {
		for i := 0; ; i++ {
			out <- i
		}
	}()
	return out
}

func firstEven() int {
	for n := range gen() {
		if n%2 == 0 {
			return n // gen 还在发送，但没人接收了
		}
	}
	return 0
}
```

修正方式是把取消信号传给上游：

```go
func gen(ctx context.Context) <-chan int {
	out := make(chan int)
	go func() {
		defer close(out)

		for i := 0; ; i++ {
			select {
			case out <- i:
			case <-ctx.Done():
				return
			}
		}
	}()
	return out
}

func firstEven(parent context.Context) int {
	ctx, cancel := context.WithCancel(parent)
	defer cancel()

	for n := range gen(ctx) {
		if n%2 == 0 {
			return n
		}
	}
	return 0
}
```

下游提前返回时执行 `cancel()`，上游的 `select` 收到取消信号后退出。

## 场景五：I/O 没有超时

外部调用如果没有超时，也可能让 goroutine 长时间卡住。

```go
func fetch(url string) (*http.Response, error) {
	return http.Get(url) // 不推荐：默认 client 没有整体超时
}
```

更好的做法是设置超时或传入 context：

```go
var client = &http.Client{
	Timeout: 3 * time.Second,
}

func fetch(ctx context.Context, url string) (*http.Response, error) {
	req, err := http.NewRequestWithContext(ctx, http.MethodGet, url, nil)
	if err != nil {
		return nil, err
	}
	return client.Do(req)
}
```

真实项目里，数据库、RPC、消息队列、锁等待都要考虑同样的问题：外部依赖不能无限等。

## 如何排查

### 1. 看 goroutine 数量趋势

瞬时数量高不一定是泄漏；关键是流量停止后能不能回落。

```go
fmt.Println(runtime.NumGoroutine())
```

测试里可以用它做粗略观察，但不要只靠它下结论，因为测试框架和运行时本身也会创建 goroutine。

### 2. 看 goroutine profile

服务里通常打开 pprof：

```go
import _ "net/http/pprof"
```

然后查看：

```sh
go tool pprof http://localhost:6060/debug/pprof/goroutine
```

或者直接看堆栈文本：

```sh
curl http://localhost:6060/debug/pprof/goroutine?debug=2
```

重点不是看有多少个 goroutine，而是看它们集中阻塞在哪些函数、哪些 channel、哪些 I/O 调用。

### 3. 用压测验证是否回落

一个实用判断方法：

- 记录服务空闲时 goroutine 基线。
- 压测一段时间。
- 停止流量。
- 等待一段时间后观察 goroutine 是否回到接近基线。

如果每轮压测后基线都上升，通常说明有泄漏或后台任务没有回收。

## 面试追问

追问参考答案：[follow-ups.md](follow-ups.md)

- 如何从 goroutine 堆栈判断泄漏卡在哪里？
- `context` 为什么不能自动杀死 goroutine？
- pipeline 下游提前退出时，上游为什么会泄漏？
- `time.Tick` 和 `time.NewTicker` 在泄漏问题上有什么区别？
- 用 goroutine 包一层慢调用，为什么可能让问题更严重？
- 如何写测试观察 goroutine 是否泄漏？
