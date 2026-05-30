# 009. nil 切片和空切片

## 问题

nil 切片和空切片有什么区别？

## 核心答案

nil 切片的值是 `nil`，长度和容量都是 0。空切片不是 nil，但长度也是 0。

多数场景下两者可以一样使用，例如 `append`、`len` 和 `range` 都正常工作。区别主要体现在和 nil 比较、JSON 序列化以及需要表达语义时。

## 代码示例

```go
var a []int
b := []int{}

fmt.Println(a == nil) // true
fmt.Println(b == nil) // false
```

## 面试追问

- JSON 中 nil slice 和 empty slice 序列化结果有什么区别？
- API 返回列表时应该返回 nil 还是空切片？
