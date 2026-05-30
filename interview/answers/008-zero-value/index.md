# 008. 零值

## 问题

Go 常见类型的零值是什么？零值可用有什么意义？

## 核心答案

Go 中变量声明后一定有零值。常见零值包括：数字是 `0`，布尔是 `false`，字符串是 `""`，指针、切片、map、channel、函数和接口是 `nil`，结构体是每个字段各自的零值。

零值可用是 Go 的重要设计。例如 `sync.Mutex` 的零值可以直接使用，`bytes.Buffer` 的零值也可以直接写入。

## 代码示例

```go
var mu sync.Mutex
mu.Lock()
mu.Unlock()

var buf bytes.Buffer
buf.WriteString("hello")
```

## 面试追问

追问参考答案：[follow-ups.md](follow-ups.md)

- nil map 可以读吗？可以写吗？
- 为什么零值可用能减少构造函数的必要性？
