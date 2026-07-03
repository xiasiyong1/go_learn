# 080. 构建 - 面试追问

## 1. `//go:build` 应该放在哪里？

放在文件开头附近，通常第一行，然后空一行再写 `package`。

```go
//go:build integration

package repo

import "testing"
```

不要放到 package 后面：

```go
package repo

//go:build integration // 错误位置，不会按预期作为构建约束
```

实际项目里建议统一格式，避免 tag 看起来写了但没生效。

## 2. build tag 和文件名平台后缀有什么关系？

它们都会影响文件是否参与构建。

```text
net_linux.go
net_windows.go
```

这些后缀由 Go 工具链自动识别，不需要写 tag。

也可以叠加：

```go
//go:build linux && cgo

package netx
```

这个文件只有在 linux 且启用 cgo 条件下才参与构建。

## 3. 如何为不同平台提供同一个函数的不同实现？

拆成多个文件，保持包名和函数签名一致。

```go
// file: home_unix.go
//go:build linux || darwin

package env

func HomeDir() string {
	return getenv("HOME")
}
```

```go
// file: home_windows.go
//go:build windows

package env

func HomeDir() string {
	return getenv("USERPROFILE")
}
```

调用方只调用：

```go
dir := env.HomeDir()
```

不要让调用方到处写 `runtime.GOOS` 判断，除非差异非常简单且不需要条件编译。

## 4. 为什么 tag 过多会增加维护成本？

每个 tag 都可能制造一条新的构建路径。

```text
default
integration
enterprise
linux
windows
linux + enterprise
windows + enterprise
```

如果 CI 只跑默认路径，其他路径可能长期坏掉。

典型风险：

- 某个 tag 下缺函数实现。
- 某个平台文件使用了不存在的系统调用。
- 集成测试 tag 下依赖环境变量，但文档没写。
- 两个 tag 组合后出现重复定义。

所以 tag 要少，用途要明确，并且关键组合要自动测试。

## 5. 如何确认某个 tag 下代码真的会编译？

直接跑目标构建。

```bash
go test -tags integration ./...
```

跨平台：

```bash
GOOS=linux GOARCH=amd64 go test ./...
GOOS=windows GOARCH=amd64 go test ./...
```

查看文件列表：

```bash
go list -json -tags integration .
```

关注输出里的：

- `GoFiles`
- `TestGoFiles`
- `IgnoredGoFiles`

这能帮助你确认某个文件到底有没有进入构建。
