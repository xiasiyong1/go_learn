# 040. Go Module - 面试追问

## 追问与参考答案

### 1. indirect 依赖是什么意思？

`indirect` 表示当前模块没有直接 import 这个依赖，但直接依赖或工具链解析需要它。它仍然会参与版本选择和校验。

### 2. `go mod tidy` 会做什么？

`go mod tidy` 会根据源码 import 增删 `go.mod` 中的依赖，并更新 `go.sum`。它会移除不再需要的依赖，也会补齐缺失的直接或间接依赖记录。
