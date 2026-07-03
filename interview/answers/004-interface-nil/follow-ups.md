# 004. interface nil 陷阱 - 面试追问

## 1. 接口值什么时候才等于 `nil`？

只有动态类型和动态值都为空时，接口值才等于 `nil`。

```go
var a any
fmt.Println(a == nil) // true

var p *int = nil
var b any = p
fmt.Println(b == nil) // false
fmt.Printf("%T %#v\n", b, b) // *int (*int)(nil)
```

面试时可以直接说：`a` 是空接口值；`b` 是装了 nil 指针的接口值。`b` 的值部分是 nil，但类型部分是 `*int`，所以整个接口不是 nil。

## 2. 为什么 `var err error = (*MyError)(nil)` 之后 `err != nil`？

因为 `error` 是接口。赋值后，`err` 的动态类型是 `*MyError`，动态值是 `nil`。

```go
type MyError struct{}

func (e *MyError) Error() string {
	return "my error"
}

func main() {
	var e *MyError = nil
	var err error = e

	fmt.Println(e == nil)   // true
	fmt.Println(err == nil) // false
	fmt.Printf("%T %#v\n", err, err)
}
```

这个问题在业务里通常表现为：

```go
func Do() error {
	var err *MyError
	return err // 错误：返回 typed nil
}
```

应该改成：

```go
func Do() error {
	var err *MyError
	if err != nil {
		return err
	}
	return nil
}
```

更好的写法是不要先声明一个 nil 的具体错误指针，没有错误时直接 `return nil`。

## 3. nil slice 放进 `any` 后，为什么 `anyValue != nil`？

因为 slice 的 nil 状态属于动态值，`any` 里仍然保存了动态类型 `[]int`。

```go
var s []int = nil
var v any = s

fmt.Println(s == nil) // true
fmt.Println(v == nil) // false
fmt.Printf("%T %#v\n", v, v) // []int []int(nil)
```

如果你需要判断接口里是不是 nil slice，可以用类型断言：

```go
func IsNilIntSlice(v any) bool {
	s, ok := v.([]int)
	return ok && s == nil
}
```

这个函数只处理 `[]int`。如果要处理任意 nil-able 类型，再考虑反射。

## 4. 类型断言后得到 nil 指针，为什么断言仍然成功？

类型断言判断的是动态类型是否匹配，不判断动态值是不是 nil。

```go
var p *MyError = nil
var err error = p

e, ok := err.(*MyError)
fmt.Println(ok)       // true
fmt.Println(e == nil) // true
```

`ok == true` 的意思是：接口里的动态类型确实是 `*MyError`。它不代表断言出来的 `e` 一定是非 nil。

所以使用断言结果时，还要根据业务判断指针是否为 nil：

```go
if e, ok := err.(*MyError); ok && e != nil {
	fmt.Println(e.Error())
}
```

## 5. 用反射判断接口底层 nil 时，为什么可能 panic？

`reflect.Value.IsNil` 只能用于 chan、func、interface、map、pointer、slice。对 int、string、struct 调用会 panic。

错误示例：

```go
func BadIsNil(v any) bool {
	return reflect.ValueOf(v).IsNil()
}

func main() {
	fmt.Println(BadIsNil(10)) // panic
}
```

安全写法：

```go
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

初学者要注意：反射能兜底处理复杂情况，但它也会绕开编译期类型检查。能用类型断言或更清晰的 API 设计解决时，不要优先上反射。

## 6. 设计返回 `error` 的函数时，如何避免 typed nil？

最重要的规则是：没有错误时直接返回 `nil`。

推荐写法：

```go
func Validate(name string) error {
	if name == "" {
		return &ValidateError{Field: "name"}
	}
	return nil
}
```

不推荐写法：

```go
func Validate(name string) error {
	var err *ValidateError
	if name == "" {
		err = &ValidateError{Field: "name"}
	}
	return err
}
```

如果这个函数未来要扩展多个错误分支，也不要为了“统一 return”牺牲语义清晰：

```go
func Validate(name string, age int) error {
	if name == "" {
		return &ValidateError{Field: "name"}
	}
	if age < 0 {
		return &ValidateError{Field: "age"}
	}
	return nil
}
```

Go 里早返回很常见，尤其适合错误处理。为了减少一行 `return nil` 而引入 typed nil，代价很高。
