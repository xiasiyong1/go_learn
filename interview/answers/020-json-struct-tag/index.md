# 020. JSON 和结构体 tag

## 问题

Go 结构体 JSON 序列化有哪些常见规则和坑？

## 核心答案

`encoding/json` 只处理导出字段。字段名可以通过 `json` tag 指定。`omitempty` 会在字段为零值时忽略该字段。

常见坑包括：未导出字段不会被序列化；`time.Time` 的零值配合 `omitempty` 不一定符合预期；nil slice 会序列化为 `null`，空 slice 会序列化为 `[]`。

## 代码示例

```go
type User struct {
	ID   int    `json:"id"`
	Name string `json:"name,omitempty"`
}
```

## 面试追问

追问参考答案：[follow-ups.md](follow-ups.md)

- 如何区分字段没传和字段传了零值？
- 自定义 JSON 序列化应该实现哪个接口？
