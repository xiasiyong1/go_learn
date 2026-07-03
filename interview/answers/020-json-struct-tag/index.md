# 020. JSON 和结构体 tag

## 问题

Go 结构体 JSON 序列化有哪些常见规则和坑？

## 先给结论

Go JSON 编解码依赖导出字段、tag、反射和接口方法。深问时要讲清楚零值、缺失字段、omitempty、自定义编解码和兼容性。

## 深入理解

### 1. 这道题真正考察什么

- 是否知道只有导出字段默认参与 JSON 编解码。
- 是否理解 `json:"name,omitempty"` 的字段名和空值省略规则。
- 是否能区分字段未传和字段传了零值。
- 是否知道 `MarshalJSON`、`UnmarshalJSON` 可以自定义行为。

### 2. 底层机制要讲清楚

- encoding/json 通过反射读取导出字段和 tag。
- `omitempty` 根据类型空值判断，0、false、空串、nil、空 slice/map 都可能被省略。
- 解码时字段不存在会保留目标字段原值；解到新结构体时就是零值。
- 指针字段可以表达三态：未传、传 null、传具体值，但需要谨慎设计。

### 3. 工程实践怎么取舍

- 对外 API 字段名应由 tag 固定，避免 Go 字段名变化影响协议。
- PATCH 或部分更新接口用指针、自定义类型或 map 区分未传和零值。
- 需要严格校验未知字段时使用 Decoder 的 `DisallowUnknownFields`。
- 性能极敏感时再考虑替代 JSON 库，先确认瓶颈。

### 4. 常见误区

- 未导出字段加 tag 也不会被 encoding/json 处理。
- 用 omitempty 后，false 或 0 被省略，导致客户端无法区分真实零值。
- 自定义 `UnmarshalJSON` 中递归调用自己导致栈溢出。
- 忽视时间、浮点数、数字精度和 unknown fields 的兼容性。

## 如何验证理解

- 为 JSON 输出写 golden test，固定字段名、null 和 [] 行为。
- 用表格测试覆盖字段缺失、字段为 null、字段为零值。
- 对兼容性敏感的 API 增加反序列化旧版本样例测试。

## 代码示例

```go
type User struct {
	ID   int    `json:"id"`
	Name string `json:"name,omitempty"`
}
```

## 面试追问

追问参考答案：[follow-ups.md](follow-ups.md)

- 如果继续追问“JSON 和结构体 tag”的底层机制，应该讲到哪些层次？
- 这个知识点在真实项目里应该如何取舍？
- 最容易踩的坑是什么？为什么这些坑不是背结论就能避免的？
- 如何用测试、工具或 profiling 验证自己的判断？
- 当数据量、并发量或团队规模变大后，这个问题会怎样升级？
