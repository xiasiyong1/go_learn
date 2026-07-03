# 036. worker pool 和 pipeline

## 问题

worker pool 和 pipeline 怎么设计？

## 先给结论

worker pool 控制并发度，pipeline 组织多阶段数据流。深问重点是背压、取消、错误传播、关闭顺序、资源上限和慢阶段处理。

## 深入理解

### 1. 这道题真正考察什么

- 是否能说明 worker 数量由 CPU、I/O、下游限额和任务耗时决定。
- 是否能解释 pipeline 每个阶段的输入、输出和关闭责任。
- 是否知道任何阶段失败都要取消其他阶段。
- 是否能处理结果收集、错误收集和 goroutine 退出。

### 2. 底层机制要讲清楚

- worker pool 通常由任务队列、固定数量 worker、结果或错误通道组成。
- pipeline 每个阶段接收上游数据、处理后发送到下游。
- 有缓冲 channel 可以吸收突发，但也可能隐藏慢消费者。
- 取消信号必须传到所有可能阻塞的发送和接收点。

### 3. 工程实践怎么取舍

- CPU 密集型 worker 数通常接近 GOMAXPROCS，I/O 密集型根据外部延迟和限额调整。
- 所有 channel 关闭应由发送方或协调者负责，避免多方 close。
- 错误发生后关闭 cancel，不再发送新任务，并等待已启动任务收尾。
- 对外部依赖设置并发上限，防止把压力转嫁给下游。

### 4. 常见误区

- 只关闭结果 channel，不取消仍在发送的上游 goroutine。
- 任务 channel 无缓冲或过大缓冲都没有结合业务评估。
- 某个阶段提前返回，导致上游阻塞发送泄漏。
- worker 数量写死，迁移环境后吞吐或下游压力异常。

## 如何验证理解

- 用压测观察不同 worker 数下的吞吐、延迟和错误率。
- 用 goroutine profile 检查失败或取消后是否还有阻塞协程。
- 写测试覆盖下游提前退出、上游报错、context 超时。

## 代码示例

```go
jobs := make(chan Job)
for i := 0; i < workerN; i++ {
	go func() {
		for job := range jobs {
			process(job)
		}
	}()
}
```

## 面试追问

追问参考答案：[follow-ups.md](follow-ups.md)

- 如果继续追问“worker pool 和 pipeline”的底层机制，应该讲到哪些层次？
- 这个知识点在真实项目里应该如何取舍？
- 最容易踩的坑是什么？为什么这些坑不是背结论就能避免的？
- 如何用测试、工具或 profiling 验证自己的判断？
- 当数据量、并发量或团队规模变大后，这个问题会怎样升级？
