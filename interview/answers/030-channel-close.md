# 030. channel 关闭

## 问题

channel 应该由谁关闭？向已关闭 channel 发送会怎样？

## 核心答案

通常由发送方关闭 channel，用于通知接收方“不会再有新数据”。接收方一般不应该关闭 channel，因为它无法确定是否还有发送方。

向已关闭 channel 发送会 panic。从已关闭 channel 接收会立即返回元素类型零值，第二个返回值 `ok` 为 false。

## 代码示例

```go
v, ok := <-ch
if !ok {
	return
}
_ = v
```

## 面试追问

- 关闭 nil channel 会怎样？
- 多个发送方时如何安全关闭 channel？
