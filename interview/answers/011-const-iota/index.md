# 011. const 和 iota

## 问题

`const` 和 `iota` 怎么用？有哪些常见坑？

## 先给结论

`const` 定义编译期常量；`iota` 是 `const` 声明块里的行计数器，从 0 开始，每一行递增。

它们适合表达稳定的常量、枚举值、位标志。但 Go 没有传统语言里封闭的 enum：即使你定义了枚举常量，变量仍然可能被赋成其他数值。

## const 是编译期常量

```go
const Name = "go"
const MaxRetry = 3
const Pi = 3.1415926
```

常量不能在运行时修改：

```go
const Timeout = 3
// Timeout = 5 // 编译错误
```

Go 还有无类型常量。它会在使用时根据上下文确定类型：

```go
const n = 10

var a int = n
var b int64 = n
var c float64 = n
```

这比直接把常量固定成某个类型更灵活。

```go
const typed int = 10

var x int64 = typed // 编译错误：不能直接把 int 常量赋给 int64 变量
```

## iota 的基本规则

`iota` 在每个 `const` 块里从 0 开始，每一行递增。

```go
const (
	A = iota
	B
	C
)

fmt.Println(A, B, C) // 0 1 2
```

省略表达式时，会复用上一行表达式：

```go
const (
	A = iota
	B // 等价于 B = iota
	C // 等价于 C = iota
)
```

新的 `const` 块会重新从 0 开始：

```go
const (
	A = iota // 0
)

const (
	B = iota // 0
)
```

## iota 是按行递增，不是按名字递增

同一行里的多个名字使用同一个 `iota` 值。

```go
const (
	A, B = iota, iota
	C, D
)

fmt.Println(A, B, C, D) // 0 0 1 1
```

空白标识符也会占用一行的 iota：

```go
const (
	_ = iota
	KB = 1 << (10 * iota)
	MB
	GB
)

fmt.Println(KB, MB, GB) // 1024 1048576 1073741824
```

这里 `_ = iota` 让 `KB` 从 `iota == 1` 开始。

## 位标志适合用 iota

```go
type Permission uint8

const (
	PermRead Permission = 1 << iota
	PermWrite
	PermExecute
)

func Has(p, flag Permission) bool {
	return p&flag != 0
}

func main() {
	p := PermRead | PermWrite
	fmt.Println(Has(p, PermRead))    // true
	fmt.Println(Has(p, PermExecute)) // false
}
```

位标志适合表达“可以组合”的状态。读、写、执行权限可以同时存在，所以适合 bit mask。

互斥状态不要乱用 bit mask。例如订单状态通常同一时间只能是一个值：

```go
type OrderStatus int

const (
	OrderPending OrderStatus = iota
	OrderPaid
	OrderShipped
	OrderClosed
)
```

## Go 的枚举不是封闭的

定义了类型和常量后，仍然可以出现非法值。

```go
type Status int

const (
	StatusPending Status = iota
	StatusRunning
	StatusDone
)

func main() {
	var s Status = 99
	fmt.Println(s) // 99，编译器不会禁止
}
```

所以公共 API 里常常需要校验：

```go
func (s Status) Valid() bool {
	switch s {
	case StatusPending, StatusRunning, StatusDone:
		return true
	default:
		return false
	}
}
```

也可以实现 `String()` 让日志更可读：

```go
func (s Status) String() string {
	switch s {
	case StatusPending:
		return "pending"
	case StatusRunning:
		return "running"
	case StatusDone:
		return "done"
	default:
		return fmt.Sprintf("Status(%d)", s)
	}
}
```

## 协议常量不要随便插入 iota

如果常量值会写入数据库、日志、消息队列、HTTP API 或跨服务协议，不要随便在中间插入 iota。

原来：

```go
const (
	RoleUser = iota
	RoleAdmin
)
```

如果后来改成：

```go
const (
	RoleGuest = iota
	RoleUser
	RoleAdmin
)
```

`RoleUser` 从 0 变成 1，`RoleAdmin` 从 1 变成 2。已经落库或对外传输的数字语义就变了。

稳定协议更推荐显式数值：

```go
const (
	RoleUser  = 1
	RoleAdmin = 2
	RoleGuest = 3
)
```

或者只在末尾追加新值，并用测试固定已有值：

```go
func TestRoleValues(t *testing.T) {
	if RoleUser != 1 || RoleAdmin != 2 {
		t.Fatal("role values changed")
	}
}
```

## 面试追问

追问参考答案：[follow-ups.md](follow-ups.md)

- 无类型常量和有类型常量有什么区别？
- `iota` 是按行递增还是按名字递增？
- 省略表达式时，`const` 块会复用什么？
- 为什么 `1 << iota` 常用于位标志？
- Go 的枚举为什么不能阻止非法值？
- 对外协议常量为什么不建议随便用 iota 插入新项？
