# 012. 包、可见性和 init

## 问题

Go 的包导入、可见性和 init 执行顺序是怎样的？

## 核心答案

Go 用包组织代码。标识符首字母大写表示导出，小写表示包内可见。

包初始化顺序大致是：先初始化被导入的包，再初始化当前包；同一个包内先初始化变量，再执行 `init` 函数。`init` 不能被手动调用。

## 代码示例

```go
package user

var DefaultName = "guest" // 可导出
var cache = map[string]int{} // 包内可见

func init() {
	cache["init"] = 1
}
```

## 面试追问

- 空白导入 `_ "pkg"` 的作用是什么？
- 一个包里可以有多个 init 函数吗？
