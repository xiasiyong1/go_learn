# 030. channel 关闭

## 问题

channel 应该由谁关闭？向已关闭 channel 发送会怎样？

## 先给结论

channel 关闭表示“不会再发送新值”，通常应由发送方关闭。关闭不是释放内存的必要动作，也不是通知发送方停止的万能手段。

## 深入理解

### 1. 这道题真正考察什么

- 是否知道向已关闭 channel 发送会 panic。
- 是否知道从已关闭 channel 接收会立刻返回零值和 ok=false。
- 是否知道关闭 nil channel 会 panic，读写 nil channel 会永久阻塞。
- 是否能处理多个发送方安全关闭问题。

### 2. 底层机制要讲清楚

- close 会唤醒等待接收者，并让后续接收立即完成。
- 关闭 channel 是广播“无更多数据”的信号，不是“取消正在发送者”的机制。
- 只有能确定不会再发送的一方，才有资格关闭 channel。
- 多个发送方需要额外协调，例如 context、done channel、WaitGroup 或单独 owner。

### 3. 工程实践怎么取舍

- 单生产者多消费者时，生产者完成后关闭数据 channel。
- 多生产者时，等待所有生产者退出后由协调者关闭 channel。
- 只需要取消信号时，关闭只读 done channel 或使用 context。
- 不需要通知接收方结束时，可以不关闭 channel，让它随对象生命周期回收。

### 4. 常见误区

- 接收方主动关闭数据 channel，导致发送方 panic。
- 多个发送方竞争 close，用 recover 掩盖设计问题。
- 把关闭 channel 当成释放资源，忽略 goroutine 是否退出。
- 从已关闭 channel 读取零值时忘记检查 ok。

## 如何验证理解

- 写测试覆盖关闭后接收、发送 panic、nil channel close。
- 用 race detector 和并发测试检查多发送方关闭路径。
- 通过 goroutine profile 验证关闭或取消后接收者退出。

## 代码示例

```go
v, ok := <-ch
if !ok {
	return
}
_ = v
```

## 面试追问

追问参考答案：[follow-ups.md](follow-ups.md)

- 如果继续追问“channel 关闭”的底层机制，应该讲到哪些层次？
- 这个知识点在真实项目里应该如何取舍？
- 最容易踩的坑是什么？为什么这些坑不是背结论就能避免的？
- 如何用测试、工具或 profiling 验证自己的判断？
- 当数据量、并发量或团队规模变大后，这个问题会怎样升级？
