# 004. interface nil 陷阱

## 问题

为什么 interface 可能出现“看起来不是 nil，但底层值是 nil”的情况？

## 先给结论

接口值不是只保存一个值。可以把它理解成两部分：

- 动态类型。
- 动态值。

只有动态类型和动态值都为空时，接口值才等于 `nil`。如果接口里装的是一个“类型为 `*T`、值为 `nil` 的指针”，那么接口的动态类型不为空，所以接口本身不等于 `nil`。

最经典的例子是：

```go
package main

import "fmt"

type MyError struct{}

func (e *MyError) Error() string {
	return "my error"
}

func returnsError() error {
	var err *MyError = nil
	return err
}

func main() {
	err := returnsError()
	fmt.Println(err == nil) // false
	fmt.Printf("%T %#v\n", err, err)
}
```

`returnsError` 返回的不是一个 nil 接口，而是一个动态类型为 `*MyError`、动态值为 `nil` 的 `error` 接口。

## 三种 nil 要分清楚

### 1. nil 接口

```go
var v any
fmt.Println(v == nil) // true
```

此时接口没有动态类型，也没有动态值。

### 2. nil 指针装进接口

```go
var p *int = nil
var v any = p

fmt.Println(p == nil) // true
fmt.Println(v == nil) // false
fmt.Printf("%T %#v\n", v, v) // *int (*int)(nil)
```

`p` 是 nil 指针，但 `v` 是一个非 nil 接口，因为它记住了动态类型 `*int`。

### 3. nil slice 装进接口

```go
var s []int = nil
var v any = s

fmt.Println(s == nil) // true
fmt.Println(v == nil) // false
fmt.Printf("%T %#v\n", v, v) // []int []int(nil)
```

slice 本身可以是 nil；但它被放入接口后，接口的动态类型是 `[]int`，所以接口不等于 nil。

## 为什么 error 最容易踩坑

`error` 是接口：

```go
type error interface {
	Error() string
}
```

如果函数签名返回 `error`，你返回一个 nil 的具体错误指针，就会把“有类型的 nil”装进接口。

错误示例：

```go
type ParseError struct {
	Line int
}

func (e *ParseError) Error() string {
	return fmt.Sprintf("parse error at line %d", e.Line)
}

func Parse(input string) error {
	var err *ParseError
	if input == "" {
		err = &ParseError{Line: 1}
	}
	return err // input 非空时，返回的是非 nil error
}
```

调用方会误判：

```go
if err := Parse("ok"); err != nil {
	// 会进入这里
	fmt.Println("failed:", err)
}
```

正确写法是没有错误就直接返回字面量 `nil`：

```go
func Parse(input string) error {
	if input == "" {
		return &ParseError{Line: 1}
	}
	return nil
}
```

如果中间确实需要用具体错误变量，也要在返回前判断：

```go
func Parse(input string) error {
	var err *ParseError
	if input == "" {
		err = &ParseError{Line: 1}
	}
	if err != nil {
		return err
	}
	return nil
}
```

## 接口方法调用和 nil receiver

接口不为 nil，不代表里面的指针一定不为 nil。

```go
type User struct {
	Name string
}

func (u *User) String() string {
	return u.Name
}

func main() {
	var u *User = nil
	var s fmt.Stringer = u

	fmt.Println(s == nil) // false
	fmt.Println(s.String()) // panic: nil pointer dereference
}
```

如果你希望 nil receiver 是合法状态，方法内部必须显式处理：

```go
func (u *User) String() string {
	if u == nil {
		return "<nil user>"
	}
	return u.Name
}
```

这个设计要谨慎。不是所有类型都应该支持 nil receiver；公共 API 要让调用方容易理解。

## 类型断言检查的是动态类型

类型断言看的是接口里的动态类型，不是变量声明时的静态类型。

```go
var p *ParseError = nil
var err error = p

pe, ok := err.(*ParseError)
fmt.Println(ok)       // true
fmt.Println(pe == nil) // true
fmt.Println(err == nil) // false
```

这段代码同时成立：

- `err != nil`，因为接口有动态类型。
- `pe == nil`，因为断言出来的具体指针值是 nil。

## 用反射判断底层 nil 要小心

有时接口入参是 `any`，你想判断里面是不是 nil 指针、nil slice、nil map 或 nil func。可以用反射，但必须先判断 kind 是否可 nil。

安全写法：

```go
package nilcheck

import "reflect"

func IsNil(v any) bool {
	if v == nil {
		return true
	}

	rv := reflect.ValueOf(v)
	switch rv.Kind() {
	case reflect.Chan, reflect.Func, reflect.Interface,
		reflect.Map, reflect.Pointer, reflect.Slice:
		return rv.IsNil()
	default:
		return false
	}
}
```

错误写法：

```go
func BadIsNil(v any) bool {
	return reflect.ValueOf(v).IsNil()
}
```

如果传入 `int`、`struct` 这类不可 nil 的值，`IsNil` 会 panic。

## 工程上怎么避免

- 返回 `error` 时，没有错误就返回字面量 `nil`。
- 不要把 nil 的具体指针直接返回给接口类型。
- 接口入参如果要判空，先判断接口本身，再判断底层值。
- 公共 API 要说明 nil receiver、nil slice、nil map 是否是合法输入。
- 能返回具体类型时不要过早返回接口；接口通常放在使用方一侧更容易控制语义。

## 面试追问

追问参考答案：[follow-ups.md](follow-ups.md)

- 接口值什么时候才等于 `nil`？
- 为什么 `var err error = (*MyError)(nil)` 之后 `err != nil`？
- nil slice 放进 `any` 后，为什么 `anyValue != nil`？
- 类型断言后得到 nil 指针，为什么断言仍然成功？
- 用反射判断接口底层 nil 时，为什么可能 panic？
- 设计返回 `error` 的函数时，如何避免 typed nil？
