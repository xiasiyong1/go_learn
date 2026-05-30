# 045. channel 底层实现 - 面试追问

## 追问与参考答案

### 1. channel 为什么不是无锁结构？

channel 需要协调缓冲区、发送队列、接收队列、关闭状态以及 goroutine 挂起唤醒，这些状态必须保持一致。runtime 使用锁来保护这些内部结构，简化正确性。

### 2. nil channel 读写为什么会永久阻塞？

nil channel 没有对应的运行时 channel 结构，发送方和接收方都没有可以配对或唤醒的对象。因此读写 nil channel 会永久阻塞，常被用来在 select 中禁用某个分支。
