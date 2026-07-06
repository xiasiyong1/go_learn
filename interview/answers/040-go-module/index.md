# 040. Go Module

## 问题

Go Module 的版本、replace 和 vendor 怎么理解？

## 先给结论

Go Module 管理模块路径、依赖版本、校验和构建输入。

核心文件：

- `go.mod`：声明模块路径、Go 版本、依赖版本、replace/exclude 等。
- `go.sum`：记录依赖模块内容的校验哈希。

常见命令：

```sh
go mod tidy
go list -m all
go mod why module/path
go mod graph
go mod vendor
```

面试要讲清楚：语义化版本、v2+ 路径规则、Minimal Version Selection、`replace` 只对当前主模块生效、vendor 适合离线或固定供应链场景。

## go.mod 和 go.sum

示例：

```go
module example.com/app

go 1.22

require github.com/gin-gonic/gin v1.10.0
```

`go.mod` 表示项目需要哪些模块和版本约束。

`go.sum` 不是“依赖列表”，而是校验文件。它记录下载过的模块内容哈希，用于验证依赖没有被篡改。

不要随便删除 `go.sum`。团队项目通常应该提交 `go.mod` 和 `go.sum`。

## indirect 是什么

```go
require golang.org/x/text v0.14.0 // indirect
```

`indirect` 表示当前模块没有直接 import 它，但依赖图里需要它。它可能来自：

- 直接依赖的依赖。
- 测试依赖。
- 某些包的工具依赖。

不要看到 `indirect` 就手删。用：

```sh
go mod tidy
```

让工具根据源码 import 自动整理。

## v2+ 路径规则

Go Module 的 v2 及以上主版本通常要体现在 module path 里：

```go
module example.com/lib/v2
```

导入也要带 `/v2`：

```go
import "example.com/lib/v2/client"
```

这是为了让 v1 和 v2 可以在同一个构建里共存，因为它们是不同的导入路径。

常见错误是升级到了 v2 版本，但 import 路径还停留在 v1。

## Minimal Version Selection

Go 使用 Minimal Version Selection。它会在依赖图中选择满足要求的版本，而不是每次自动取最新版本。

如果你的模块要求：

```go
require example.com/lib v1.2.0
```

另一个依赖要求：

```go
require example.com/lib v1.5.0
```

最终会选择 `v1.5.0`。但不会因为有 `v1.9.0` 就自动升级到 `v1.9.0`。

查看最终版本：

```sh
go list -m all
```

升级依赖：

```sh
go get example.com/lib@v1.9.0
go mod tidy
```

## replace

本地联调：

```go
replace example.com/lib => ../lib
```

临时 fork：

```go
replace example.com/lib => github.com/your/lib v1.2.3-fix
```

关键点：

- `replace` 只在当前主模块生效。
- 下游依赖你的库时，不会继承你的 replace。
- 发布库前要检查临时 replace 是否应该移除。

如果应用仓库需要 replace，可以接受；如果公共库提交了指向本地路径的 replace，通常会让使用者困惑。

## vendor

生成 vendor：

```sh
go mod vendor
```

使用 vendor 构建：

```sh
go build -mod=vendor ./...
```

vendor 适合：

- 离线构建。
- 固定供应链输入。
- 公司内部审计依赖源码。

代价是仓库变大，依赖升级 diff 变重。是否使用 vendor 要团队统一约定。

## 常用排查命令

看最终依赖版本：

```sh
go list -m all
```

看某依赖为什么被引入：

```sh
go mod why github.com/pkg/errors
```

看依赖图：

```sh
go mod graph
```

整理依赖：

```sh
go mod tidy
```

干净环境构建能验证 `replace`、私有代理、vendor 假设是否成立。

## 面试追问

追问参考答案：[follow-ups.md](follow-ups.md)

- `go.mod` 和 `go.sum` 分别负责什么？
- `// indirect` 能不能手动删除？
- v2 及以上版本为什么通常要改 module path？
- Minimal Version Selection 和“每次取最新”有什么区别？
- `replace` 的生效范围是什么？
- vendor 适合什么场景，代价是什么？
