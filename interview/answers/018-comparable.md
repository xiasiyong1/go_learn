# 018. 可比较类型

## 问题

Go 中哪些类型可以比较？`comparable` 表示什么？

## 核心答案

可比较类型可以使用 `==` 和 `!=`。基础类型、指针、channel、接口以及只包含可比较字段的结构体通常可比较。

slice、map、function 不可比较，只能和 nil 比较。

`comparable` 是泛型中的预声明约束，表示类型参数必须支持 `==` 和 `!=`。

## 代码示例

```go
func Index[T comparable](items []T, target T) int {
	for i, item := range items {
		if item == target {
			return i
		}
	}
	return -1
}
```

## 面试追问

- interface 比较时什么时候会 panic？
- 为什么 map 的 key 必须可比较？
