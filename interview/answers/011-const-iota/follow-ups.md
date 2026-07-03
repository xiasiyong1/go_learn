# 011. const 和 iota - 面试追问

## 1. 无类型常量和有类型常量有什么区别？

无类型常量会在使用时根据上下文确定类型。

```go
const n = 10

var a int = n
var b int64 = n
var c float64 = n
```

有类型常量已经固定类型：

```go
const n int = 10

var a int = n
// var b int64 = n // 编译错误
```

无类型常量还有更高精度，常用于数值表达式：

```go
const x = 1.0 / 3.0
var f float64 = x
```

面试回答可以说：无类型常量提高了常量表达式的灵活性，但一旦赋给变量，就会变成具体类型的值。

## 2. `iota` 是按行递增还是按名字递增？

按 const 声明块里的行递增，不是按名字递增。

```go
const (
	A, B = iota, iota
	C, D
	E
)

fmt.Println(A, B) // 0 0
fmt.Println(C, D) // 1 1
fmt.Println(E)    // 2
```

同一行的多个常量共享同一个 `iota` 值。

空白标识符所在行也会消耗一个 iota：

```go
const (
	_ = iota
	One
	Two
)

fmt.Println(One, Two) // 1 2
```

## 3. 省略表达式时，`const` 块会复用什么？

会复用上一行的表达式，但其中的 `iota` 值会按当前行重新计算。

```go
const (
	A = iota * 10
	B
	C
)

fmt.Println(A, B, C) // 0 10 20
```

`B` 不是直接复制 `A` 的值，而是复用表达式 `iota * 10`，此时 `iota == 1`。

更复杂一点：

```go
const (
	A = iota
	B = 100
	C
	D = iota
)

fmt.Println(A, B, C, D) // 0 100 100 3
```

`C` 复用上一行表达式 `100`，所以还是 100。`D` 显式使用当前行的 `iota`，所以是 3。

## 4. 为什么 `1 << iota` 常用于位标志？

因为它可以生成 1、2、4、8 这样的二进制位，每个常量占一个独立 bit，适合组合。

```go
type Flag uint8

const (
	Read Flag = 1 << iota
	Write
	Execute
)

func Has(flags, target Flag) bool {
	return flags&target != 0
}

func main() {
	flags := Read | Write
	fmt.Println(Has(flags, Read))    // true
	fmt.Println(Has(flags, Execute)) // false
}
```

如果状态互斥，不要用 bit mask。

```go
type Status int

const (
	Pending Status = iota
	Running
	Done
)
```

订单不应该同时是 Pending 和 Done，所以普通枚举更清楚。

## 5. Go 的枚举为什么不能阻止非法值？

Go 没有封闭 enum。自定义类型加常量只能提高可读性，不能保证变量只取这些常量。

```go
type Status int

const (
	Pending Status = iota
	Running
	Done
)

var s Status = 99 // 可以编译
```

所以需要在边界处校验：

```go
func (s Status) Valid() bool {
	switch s {
	case Pending, Running, Done:
		return true
	default:
		return false
	}
}
```

如果来自 JSON、数据库、RPC 的值要转成枚举，必须处理未知值。

```go
func ParseStatus(n int) (Status, error) {
	s := Status(n)
	if !s.Valid() {
		return 0, fmt.Errorf("invalid status %d", n)
	}
	return s, nil
}
```

## 6. 对外协议常量为什么不建议随便用 iota 插入新项？

因为 `iota` 的值依赖声明顺序。中间插入一行，会改变后续所有值。

```go
const (
	User = iota
	Admin
)
```

此时 `User == 0`，`Admin == 1`。

后来改成：

```go
const (
	Guest = iota
	User
	Admin
)
```

`User` 变成 1，`Admin` 变成 2。旧数据、旧客户端、旧日志都会被错误解释。

稳定协议建议显式赋值：

```go
const (
	User  = 1
	Admin = 2
	Guest = 3
)
```

并用测试固定：

```go
func TestRoleValues(t *testing.T) {
	if User != 1 || Admin != 2 || Guest != 3 {
		t.Fatal("role values changed")
	}
}
```

面试里可以总结：内部临时代码用 iota 很方便；跨边界、落库、对外协议要优先稳定性。
