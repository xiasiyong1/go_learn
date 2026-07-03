# 040. Go Module

## 问题

Go Module 的版本、replace 和 vendor 怎么理解？

## 先给结论

Go Module 管理模块路径、语义化版本、依赖图和校验。深入理解要覆盖 minimal version selection、replace、vendor、indirect 和可复现构建。

## 深入理解

### 1. 这道题真正考察什么

- 是否知道 module path 是导入路径的一部分。
- 是否理解语义化版本，尤其 v2+ 需要路径后缀。
- 是否知道 `go.mod` 和 `go.sum` 的职责不同。
- 是否能解释 `replace`、`exclude`、`vendor` 和 indirect。

### 2. 底层机制要讲清楚

- Go 使用 Minimal Version Selection，选择依赖图中要求的最低可用最高版本，而不是每次取最新。
- `go.mod` 描述直接和间接依赖要求，`go.sum` 记录校验哈希。
- v2 及以上主版本通常需要在 module path 中包含 `/v2`。
- `replace` 只在当前主模块生效，用于本地开发、临时替换或 fork 修复。

### 3. 工程实践怎么取舍

- 提交 `go.mod` 和 `go.sum`，保证团队构建一致。
- 使用 `go mod tidy` 清理未使用依赖并补齐缺失依赖。
- 本地联调可用 replace，但发布前要确认是否应该移除。
- 需要离线或供应链固定时使用 vendor，并明确构建参数。

### 4. 常见误区

- 误删 go.sum，导致校验和重新变化或 CI 失败。
- 在库模块中提交临时 replace，影响使用者预期。
- 升级 v2 依赖但导入路径没改，导致版本不符合规范。
- 看到 indirect 就手动删除，未理解它可能是测试或间接依赖需要。

## 如何验证理解

- 用 `go list -m all` 查看最终模块版本。
- 用 `go mod why` 分析某依赖为什么被引入。
- 用 `go mod graph` 查看依赖图。
- 用干净环境跑构建，验证 replace/vendor 假设。

## 代码示例

```go
replace example.com/lib => ../lib
```

## 面试追问

追问参考答案：[follow-ups.md](follow-ups.md)

- 如果继续追问“Go Module”的底层机制，应该讲到哪些层次？
- 这个知识点在真实项目里应该如何取舍？
- 最容易踩的坑是什么？为什么这些坑不是背结论就能避免的？
- 如何用测试、工具或 profiling 验证自己的判断？
- 当数据量、并发量或团队规模变大后，这个问题会怎样升级？
