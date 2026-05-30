# 028. goroutine 和 channel 基础 - 面试追问

## 追问与参考答案

### 1. goroutine 和系统线程有什么区别？

goroutine 由 Go runtime 调度，初始栈小，可以大量创建；系统线程由操作系统调度，创建和上下文切换成本更高。Go 调度器会把许多 goroutine 复用到较少线程上运行。

### 2. channel 是否一定比 mutex 更好？

不是。channel 适合表达数据流、任务队列和生命周期信号；mutex 适合保护共享状态和复杂不变式。工程里应按问题模型选择，而不是固定偏好某一种。
