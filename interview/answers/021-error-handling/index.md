# 021. error 处理

## 问题

Go 为什么推荐显式返回 error？`errors.Is/As` 和 wrapping 怎么用？

## 核心答案

Go 推荐把错误作为普通返回值显式处理，这让控制流清楚，也避免异常式跳转隐藏失败路径。

Go 1.13 之后可以用 `%w` 包装错误，用 `errors.Is` 判断错误链中是否包含某个哨兵错误，用 `errors.As` 提取某种具体错误类型。

## 代码示例

```go
if err != nil {
	return fmt.Errorf("load config: %w", err)
}
```

## 面试追问

追问参考答案：[follow-ups.md](follow-ups.md)

- 什么时候应该定义哨兵错误？
- 为什么不要只返回字符串错误？
