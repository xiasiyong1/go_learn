# 030. channel 关闭 - 面试追问

## 追问与参考答案

### 1. 关闭 nil channel 会怎样？

关闭 nil channel 会 panic。对 nil channel 的发送和接收会永久阻塞，但 close nil channel 不是阻塞，而是直接 panic。

### 2. 多个发送方时如何安全关闭 channel？

多个发送方时不要让每个发送方自行 close。通常用额外的协调 goroutine 在所有发送方退出后统一关闭，或者用 context 通知退出，避免重复关闭和发送到已关闭 channel。
