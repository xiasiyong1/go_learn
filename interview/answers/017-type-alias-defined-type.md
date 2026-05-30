# 017. 类型别名和新类型

## 问题

类型别名和新类型有什么区别？

## 核心答案

类型别名使用 `type A = B`，A 和 B 是同一个类型，常用于迁移和兼容。

新类型使用 `type A B`，A 是一个独立类型，底层类型是 B，但方法集合、类型检查都独立。

## 代码示例

```go
type MyString string
type AliasString = string
```

`MyString` 不能直接当作 `string` 使用，需要显式转换；`AliasString` 就是 `string`。

## 面试追问

- 为什么给基础类型定义新类型有价值？
- 类型别名适合在哪些重构场景使用？
