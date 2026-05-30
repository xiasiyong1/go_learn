# 036. worker pool 和 pipeline

## 问题

worker pool 和 pipeline 怎么设计？

## 核心答案

worker pool 用固定数量的 worker 消费任务，核心目的是限制并发度，避免无限 goroutine 压垮资源。

pipeline 把处理流程拆成多个阶段，每个阶段通过 channel 传递数据。设计时要考虑取消、错误传播、关闭 channel 和背压。

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

- worker 数量怎么确定？
- pipeline 中某个阶段失败后如何取消其他阶段？
