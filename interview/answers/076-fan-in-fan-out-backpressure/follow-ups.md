# 076. 并发模式 - 面试追问

## 1. fan-out 和 worker pool 是一回事吗？

概念有重叠，但侧重点不同。fan-out 强调把任务分发到多个并行执行单元；worker pool 强调固定数量 worker 和任务队列，用来控制并发上限。

```go
jobs := make(chan Job)

for i := 0; i < 5; i++ {
	go func() {
		for job := range jobs {
			process(job)
		}
	}()
}
```

这既是 fan-out，也是一个简单 worker pool。

如果只是把一份数据广播给多个下游，那不是这个 worker pool 模型，因为每个 worker 都会收到同一份数据；这里的多个 worker 是竞争消费任务。

## 2. fan-in 的输出 channel 应该由谁关闭？

应该由负责协调所有发送者的 goroutine 关闭，而不是某个单独发送者。

错误示例：

```go
go func() {
	defer close(out) // 错：这个输入结束不代表其他输入结束
	for v := range in1 {
		out <- v
	}
}()
```

正确写法：

```go
var wg sync.WaitGroup
for _, in := range inputs {
	in := in
	wg.Add(1)
	go func() {
		defer wg.Done()
		for v := range in {
			out <- v
		}
	}()
}

go func() {
	wg.Wait()
	close(out)
}()
```

记住原则：多个发送者时，close 要由能确认“所有发送者都结束”的协调者执行。

## 3. 消费者只要第一个结果时，如何避免上游泄漏？

如果消费者只读一个结果就返回，上游继续发送会阻塞。

```go
result := <-results
return result // 上游可能还在发
```

需要取消上游：

```go
ctx, cancel := context.WithCancel(parent)
defer cancel()

select {
case r := <-results:
	return r, nil
case <-ctx.Done():
	return Result{}, ctx.Err()
}
```

同时，上游发送必须监听 context：

```go
select {
case results <- r:
case <-ctx.Done():
	return
}
```

如果上游发送点不监听 context，消费者 cancel 了也没用。

## 4. channel 缓冲区越大越好吗？

不是。缓冲区只是在短时间内吸收生产和消费速度差。

```go
jobs := make(chan Job, 1000)
```

大缓冲的代价：

- 更多内存占用。
- 请求在队列里等更久，尾延迟变高。
- 下游慢的问题被隐藏，失败来得更晚。

如果消费者长期处理 100/s，生产者长期 1000/s，任何有限缓冲都会满。真正要做的是限流、丢弃、扩容、降级或增加消费者能力。

## 5. 如何判断 worker 数量设置得是否合理？

看任务类型和指标。

CPU 密集任务：

```go
workerN := runtime.GOMAXPROCS(0)
```

I/O 密集任务：

```go
workerN := 50 // 需要结合下游连接池、延迟、QPS 测出来
```

观察指标：

- jobs 队列长度是否长期接近容量。
- worker 是否经常空闲。
- 下游错误率和 P99 是否随 worker 增加而恶化。
- 本服务 goroutine、内存、CPU 是否明显上升。

worker 数不是配置越大越好，而是要让系统在吞吐、延迟、下游保护之间达到平衡。
