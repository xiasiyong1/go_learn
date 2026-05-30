# 027. context

## 问题

`context.Context` 应该怎么用？常见误用有哪些？

## 核心答案

`context` 用于在请求链路中传递取消信号、超时截止时间和少量请求级元数据。

常见规则：函数通常把 `context.Context` 作为第一个参数；不要把 context 存在结构体里；不要用 context 传业务参数；创建带取消的 context 后要调用 cancel。

## 代码示例

```go
ctx, cancel := context.WithTimeout(context.Background(), time.Second)
defer cancel()

err := callRemote(ctx)
```

## 面试追问

追问参考答案：[follow-ups.md](follow-ups.md)

- `WithCancel`、`WithTimeout` 和 `WithDeadline` 有什么区别？
- 为什么要调用 cancel？
