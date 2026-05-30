# 033. WaitGroup

## 问题

`WaitGroup` 怎么用？常见错误有哪些？

## 核心答案

`WaitGroup` 用于等待一组 goroutine 完成。启动 goroutine 前调用 `Add`，goroutine 结束时调用 `Done`，等待方调用 `Wait`。

常见错误包括：在 goroutine 内部才 `Add` 导致 Wait 提前返回；`Done` 调用次数和 `Add` 不匹配；复制已经使用过的 WaitGroup。

## 代码示例

```go
var wg sync.WaitGroup
for _, job := range jobs {
	wg.Add(1)
	go func(job Job) {
		defer wg.Done()
		process(job)
	}(job)
}
wg.Wait()
```

## 面试追问

追问参考答案：[follow-ups.md](follow-ups.md)

- WaitGroup 能不能复用？
- 如何收集多个 goroutine 的错误？
