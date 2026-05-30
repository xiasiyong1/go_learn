# 058. unsafe

## 问题

`unsafe` 能做什么？使用边界是什么？

## 核心答案

`unsafe` 可以绕过 Go 类型安全做底层内存操作，例如指针转换、结构体布局访问、零拷贝转换等。

使用 unsafe 会依赖运行时和编译器实现细节，容易破坏 GC 正确性、内存对齐和可移植性。除非有明确性能证据或底层系统需求，否则不应该使用。

## 代码示例

```go
size := unsafe.Sizeof(MyStruct{})
```

## 面试追问

追问参考答案：[follow-ups.md](follow-ups.md)

- `unsafe.Pointer` 和 `uintptr` 有什么区别？
- 为什么 unsafe 代码要特别注意对象生命周期？
