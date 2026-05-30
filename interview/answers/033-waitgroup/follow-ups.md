# 033. WaitGroup - 面试追问

## 追问与参考答案

### 1. WaitGroup 能不能复用？

可以在上一轮 `Wait` 返回后复用，但不能在还有 goroutine 使用时和下一轮混在一起。更清晰的做法是每批任务使用新的 WaitGroup。

### 2. 如何收集多个 goroutine 的错误？

标准库 WaitGroup 只等待完成，不收集错误。可以用 channel 收集错误，或使用 `errgroup.Group`，它能结合 context 在某个 goroutine 出错后取消其他任务。
