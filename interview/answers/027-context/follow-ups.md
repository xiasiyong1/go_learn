# 027. context - 面试追问

## 追问与参考答案

### 1. `WithCancel`、`WithTimeout` 和 `WithDeadline` 有什么区别？

`WithCancel` 只提供手动取消，`WithTimeout` 在一段时间后自动取消，`WithDeadline` 在指定时间点取消。`WithTimeout` 本质上是基于当前时间计算 deadline。

### 2. 为什么要调用 cancel？

调用 cancel 可以释放 context 内部的定时器和相关资源，并尽快通知下游退出。即使已经超时，调用 cancel 也是推荐习惯，通常用 `defer cancel()`。
