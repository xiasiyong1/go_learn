# 040. Go Module - 面试追问

## 1. `go.mod` 和 `go.sum` 分别负责什么？

`go.mod` 描述模块和依赖要求。

```go
module example.com/app

go 1.22

require github.com/gin-gonic/gin v1.10.0
```

`go.sum` 记录模块内容校验哈希，用来验证下载的依赖是否匹配。

团队项目通常都要提交：

```text
go.mod
go.sum
```

不要把 `go.sum` 当缓存随便删。它是可复现和安全校验的一部分。

## 2. `// indirect` 能不能手动删除？

不建议手动删。让 `go mod tidy` 根据代码整理。

```go
require golang.org/x/text v0.14.0 // indirect
```

它表示当前模块没有直接 import，但依赖图需要它。

正确操作：

```sh
go mod tidy
```

如果 tidy 后它还在，说明仍然需要。可能是直接依赖的依赖，也可能是测试路径需要。

## 3. v2 及以上版本为什么通常要改 module path？

因为 Go Module 用导入路径区分不同主版本。v2+ 通常需要 `/v2` 后缀。

```go
module example.com/lib/v2
```

导入：

```go
import "example.com/lib/v2/client"
```

这样 v1 和 v2 可以同时存在：

```go
import old "example.com/lib/client"
import new "example.com/lib/v2/client"
```

如果升级 v2 但路径不变，通常会遇到版本或导入不匹配问题。

## 4. Minimal Version Selection 和“每次取最新”有什么区别？

MVS 会选择依赖图中要求的最低可用最高版本，不会自动追最新。

如果 A 要求：

```text
lib v1.2.0
```

B 要求：

```text
lib v1.5.0
```

最终选择 `v1.5.0`。即使存在 `v1.9.0`，也不会自动升级。

查看最终版本：

```sh
go list -m all
```

主动升级：

```sh
go get example.com/lib@v1.9.0
go mod tidy
```

## 5. `replace` 的生效范围是什么？

只在当前主模块生效。

```go
replace example.com/lib => ../lib
```

它适合本地联调。但如果你是库作者，下游使用你的库时，不会继承你的 replace。

发布前要检查：

```sh
go list -m all
```

公共库里提交本地路径 replace 通常是不合适的。

## 6. vendor 适合什么场景，代价是什么？

适合：

- 离线构建。
- 固定依赖源码。
- 供应链审计。
- 网络访问受限的构建环境。

生成：

```sh
go mod vendor
```

使用：

```sh
go build -mod=vendor ./...
```

代价：

- 仓库体积变大。
- 依赖升级 diff 变大。
- 要维护 vendor 与 go.mod 一致。

是否使用 vendor 应该是团队构建策略，而不是个人临时选择。
