# 052. 减少内存分配

## 问题

如何减少 Go 程序的内存分配？

## 核心答案

先用 benchmark 和 pprof 找到真实分配热点，再做针对性优化。

常见手段包括：预分配 slice/map 容量；复用 buffer；避免不必要的字符串和字节切片转换；减少接口装箱；避免在热点路径创建临时对象；合理控制对象生命周期。

## 代码示例

```go
items := make([]Item, 0, expected)
```

## 面试追问

追问参考答案：[follow-ups.md](follow-ups.md)

- 为什么过早优化会让代码变复杂？
- `strings.Builder` 和 `bytes.Buffer` 怎么选？
