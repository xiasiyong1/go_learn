# 080. 构建

## 问题

Go build tags 和按平台条件编译应该怎么用？

## 先给结论

build tag 是文件级别的构建约束，用来决定某个 `.go` 文件是否参与本次构建。它常用于平台差异、可选功能、集成测试、工具代码。好处是边界清楚，坏处是会制造多条构建路径；每条路径都要能编译、能测试。

## 1. `//go:build` 写在文件开头

示例：只有指定 `integration` tag 时才编译这个测试文件。

```go
//go:build integration

package user_test

import "testing"

func TestWithRealDB(t *testing.T) {
	// connect real db
}
```

运行：

```bash
go test -tags integration ./...
```

不带 tag 时，这个文件不会进入构建。

`//go:build` 必须在文件开头附近，通常放第一行，后面空一行再写 `package`。

## 2. 平台后缀也是构建约束

文件名可以表达平台。

```text
path_unix.go
path_windows.go
```

也可以更具体：

```text
syscall_linux_amd64.go
syscall_darwin_arm64.go
```

Go 工具链会根据 `GOOS`、`GOARCH` 自动选择。

```bash
GOOS=linux GOARCH=amd64 go test ./...
GOOS=windows GOARCH=amd64 go test ./...
```

平台文件应提供同一组函数，让调用方不关心平台差异。

```go
// path_unix.go
package pathx

func defaultConfigPath() string {
	return "/etc/app/config.yaml"
}
```

```go
// path_windows.go
package pathx

func defaultConfigPath() string {
	return `C:\ProgramData\App\config.yaml`
}
```

## 3. tag 是文件级别，不是语句级别

build tag 决定整个文件是否参与构建，不能只控制某个函数或某几行。

如果你需要同一个函数在不同 tag 下有不同实现，就拆文件。

```text
feature_enabled.go
feature_disabled.go
```

```go
//go:build feature

package feature

func Enabled() bool { return true }
```

```go
//go:build !feature

package feature

func Enabled() bool { return false }
```

两个文件提供相同 API，调用方不用写条件判断。

## 4. `_test.go` 只参与测试构建

`xxx_test.go` 文件不会进入普通 `go build`，只在 `go test` 时参与。

```go
// user_test.go
package user

func TestCreate(t *testing.T) {}
```

集成测试可以组合 `_test.go` 和 build tag。

```go
//go:build integration

package user

func TestRealDatabase(t *testing.T) {}
```

这样普通单测不会依赖真实数据库。

## 5. 常见坑：某个 tag 下缺实现

例如有调用方依赖：

```go
func Run() {
	startWorker()
}
```

但 `startWorker` 只在某个 tag 文件里定义：

```go
//go:build worker

package app

func startWorker() {}
```

不带 `worker` tag 构建时就会失败。每组条件编译文件必须保证所有目标构建条件下 API 完整。

## 6. 如何验证

常规测试：

```bash
go test ./...
```

关键 tag 测试：

```bash
go test -tags integration ./...
go test -tags feature ./...
```

平台构建：

```bash
GOOS=linux GOARCH=amd64 go test ./...
GOOS=darwin GOARCH=arm64 go test ./...
```

检查某个文件是否参与构建，可以用：

```bash
go list -json .
```

查看 `GoFiles`、`IgnoredGoFiles`。

## 7. 面试时怎么答

可以这样回答：

- build tag 是文件级构建约束，控制某个文件是否参与构建。
- `//go:build` 放在文件开头，`go build -tags` 或 `go test -tags` 启用自定义 tag。
- `_linux.go`、`_windows.go`、`_amd64.go` 是平台约束。
- `_test.go` 只参与测试构建。
- 条件编译文件应提供一致 API，避免某些 tag 下缺实现。
- tag 组合要在 CI 中测试，否则很容易只在发布时才发现失败。

## 面试追问

追问参考答案：[follow-ups.md](follow-ups.md)

- `//go:build` 应该放在哪里？
- build tag 和文件名平台后缀有什么关系？
- 如何为不同平台提供同一个函数的不同实现？
- 为什么 tag 过多会增加维护成本？
- 如何确认某个 tag 下代码真的会编译？
