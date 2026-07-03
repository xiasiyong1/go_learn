# 009. nil 切片和空切片

## 问题

nil 切片和空切片有什么区别？

## 先给结论

nil 切片和空切片长度都为 0，也都可以 append，但它们的底层状态和序列化表现可能不同。面试要能说清楚语义、JSON 差异和 API 约定。

## 深入理解

### 1. 这道题真正考察什么

- 是否知道 `nil` slice 的指针为空，empty slice 通常有非 nil 切片头指针。
- 是否知道二者 `len` 和 `cap` 都可能为 0，遍历和 append 行为一致。
- 是否知道 JSON 中 nil slice 编码为 `null`，empty slice 编码为 `[]`。
- 是否能根据 API 语义选择返回 nil 还是空切片。

### 2. 底层机制要讲清楚

- 切片是描述符；nil 切片描述符中的数据指针为 nil。
- 空切片可以来自字面量 `[]T{}` 或 `make([]T, 0)`，它表示“有一个空集合”。
- append 对 nil slice 会分配新底层数组，这也是 nil slice 常被当作自然零值使用的原因。
- 序列化框架会区分 nil 和 empty，因为它们表达的状态不同。

### 3. 工程实践怎么取舍

- 内部函数返回结果列表时，nil slice 往往更简单，调用方可以直接 range 和 append。
- 对外 JSON API 通常倾向返回 `[]` 表示空列表，避免前端额外处理 null。
- 需要表达“未加载”和“已加载但为空”时，nil 和 empty 可以承载不同语义。
- 团队应统一接口约定，不要每个接口各自选择。

### 4. 常见误区

- 为了避免 nil，写很多无意义的 `make([]T, 0)`。
- 对外接口把 null 和 [] 混用，导致客户端兼容问题。
- 用 `s == nil` 判断业务是否有数据，而不是用 `len(s)`。
- 忽视 `omitempty` 对 nil 和 empty slice 的编码影响。

## 如何验证理解

- 写单测覆盖 `json.Marshal(nilSlice)` 和 `json.Marshal(emptySlice)`。
- 用 `len(s) == 0` 判断是否为空集合。
- 在 API 契约测试中固定空列表的 JSON 形式。

## 代码示例

```go
var a []int
b := []int{}

fmt.Println(a == nil) // true
fmt.Println(b == nil) // false
```

## 面试追问

追问参考答案：[follow-ups.md](follow-ups.md)

- 如果继续追问“nil 切片和空切片”的底层机制，应该讲到哪些层次？
- 这个知识点在真实项目里应该如何取舍？
- 最容易踩的坑是什么？为什么这些坑不是背结论就能避免的？
- 如何用测试、工具或 profiling 验证自己的判断？
- 当数据量、并发量或团队规模变大后，这个问题会怎样升级？
