# 005. goroutine 泄漏 - 面试追问

## 追问与参考答案

### 1. `context` 应该由谁创建，谁取消？

通常由调用方创建 context，因为调用方知道请求或任务的生命周期；谁创建带取消能力的 context，谁负责调用 cancel。被调用方只接收 context 并响应取消，不应该长期保存。

### 2. 如何用 `pprof` 排查 goroutine 泄漏？

可以通过 `net/http/pprof` 或 `go tool pprof` 查看 goroutine profile，重点看大量重复栈停在哪里。常见泄漏点是 channel 收发阻塞、锁等待、网络 I/O 等没有退出路径的位置。
