# 042. GMP 调度器 - 面试追问

## 追问与参考答案

### 1. GOMAXPROCS 控制什么？

`GOMAXPROCS` 控制同时执行 Go 代码的 P 的数量，也就是 Go 层面的并行度上限。它不等于 goroutine 数量，也不直接等于系统线程数量。

### 2. goroutine 阻塞系统调用时调度器如何处理？

当 goroutine 进入阻塞系统调用时，执行它的 M 可能暂时和 P 分离，runtime 会把 P 交给其他 M 继续运行 Go 代码。系统调用返回后，原 goroutine 再重新进入可运行状态等待调度。
