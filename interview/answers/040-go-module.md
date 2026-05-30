# 040. Go Module

## 问题

Go Module 的版本、replace 和 vendor 怎么理解？

## 核心答案

Go Module 用 `go.mod` 描述模块路径和依赖版本，用 `go.sum` 记录依赖校验信息。

语义化版本中，v2 及以上主版本通常需要体现在模块路径里，例如 `/v2`。

`replace` 用于把某个依赖替换成本地路径或其他版本，常用于本地调试。`vendor` 会把依赖复制到项目中，适合某些离线或强管控构建场景。

## 代码示例

```go
replace example.com/lib => ../lib
```

## 面试追问

- indirect 依赖是什么意思？
- `go mod tidy` 会做什么？
