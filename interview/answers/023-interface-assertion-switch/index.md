# 023. 类型断言和 type switch

## 问题

空接口、类型断言和 type switch 分别适合什么场景？

## 核心答案

`interface{}` 或 `any` 可以接收任意类型，但会丢失静态类型信息。

类型断言用于从接口值中取出具体类型，推荐使用 `v, ok := x.(T)` 避免 panic。

type switch 适合对多个可能类型做分支处理。

## 代码示例

```go
switch v := x.(type) {
case string:
	fmt.Println("string", v)
case int:
	fmt.Println("int", v)
default:
	fmt.Println("unknown")
}
```

## 面试追问

追问参考答案：[follow-ups.md](follow-ups.md)

- `any` 和 `interface{}` 有区别吗？
- 接口设计为什么应该尽量小？
