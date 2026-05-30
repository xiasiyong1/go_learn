# 036. worker pool 和 pipeline - 面试追问

## 追问与参考答案

### 1. worker 数量怎么确定？

worker 数量通常由瓶颈资源决定。CPU 密集任务接近 CPU 核数，I/O 密集任务可以更高，但要受下游连接数、限流、内存和延迟目标约束，最终应通过压测确定。

### 2. pipeline 中某个阶段失败后如何取消其他阶段？

应使用 context 或 done channel 广播取消信号，各阶段在 select 中监听取消，并在退出时关闭自己负责的输出 channel。错误可以通过单独 error channel 或 errgroup 向上传递。
