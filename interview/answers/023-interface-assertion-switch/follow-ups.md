# 023. 类型断言和 type switch - 面试追问

## 1. `any` 和 `interface{}` 是什么关系？

`any` 是 `interface{}` 的别名。

```go
var a any = 1
var b interface{} = "go"

fmt.Printf("%T %T\n", a, b)
```

它们表达的是同一件事：可以装任意类型的值。

`any` 只是更直观，表示“任意值”。但不要因为名字更短就滥用。业务代码能用具体类型，就优先用具体类型。

## 2. 单返回值断言和双返回值断言有什么区别？

单返回值断言失败会 panic：

```go
var v any = "go"
n := v.(int) // panic
_ = n
```

双返回值断言失败不会 panic：

```go
var v any = "go"
n, ok := v.(int)
if !ok {
	fmt.Println("not int")
	return
}
fmt.Println(n)
```

普通输入校验和业务分支应该用双返回值形式。单返回值形式只适合你已经能证明类型一定正确的内部代码。

## 3. 类型断言目标是接口时，检查的是什么？

检查动态值是否实现目标接口。

```go
type Stringer interface {
	String() string
}

func Format(v any) string {
	s, ok := v.(Stringer)
	if ok {
		return s.String()
	}
	return fmt.Sprint(v)
}
```

如果传入的是 `time.Duration`，它实现了 `String()`，断言就成功。

这和断言具体类型不同：

```go
if d, ok := v.(time.Duration); ok {
	return d.String()
}
```

接口断言更强调能力，具体类型断言更强调类型身份。

## 4. type switch 适合什么场景，什么时候说明设计有问题？

适合处理有限、明确的动态类型集合：

```go
func ToString(v any) string {
	switch x := v.(type) {
	case string:
		return x
	case []byte:
		return string(x)
	case fmt.Stringer:
		return x.String()
	default:
		return fmt.Sprint(x)
	}
}
```

如果 type switch 分支不断增长，通常说明缺少抽象。

```go
switch v.(type) {
case EmailMessage:
case SMSMessage:
case PushMessage:
case WebhookMessage:
}
```

可以考虑接口：

```go
type Sender interface {
	Send(context.Context) error
}
```

让每个类型自己实现行为，而不是中心化地判断所有类型。

## 5. typed nil 断言成功后为什么还可能拿到 nil 指针？

因为断言检查的是动态类型是否匹配，不保证动态值非 nil。

```go
type MyError struct{}

func (e *MyError) Error() string {
	return "my error"
}

var e *MyError = nil
var err error = e

me, ok := err.(*MyError)
fmt.Println(ok)        // true
fmt.Println(me == nil) // true
```

`err` 的动态类型是 `*MyError`，所以断言成功；动态值是 nil，所以 `me == nil`。

使用断言结果前，要根据业务判断 nil 是否允许。

## 6. 为什么业务核心逻辑不应该到处使用 `any`？

因为它丢掉了编译期类型检查。

不推荐：

```go
func Calculate(input any) int {
	m := input.(map[string]any)
	price := m["price"].(int)
	count := m["count"].(int)
	return price * count
}
```

任何字段名或类型错误都会在运行时 panic。

推荐明确建模：

```go
type OrderLine struct {
	Price int
	Count int
}

func Calculate(line OrderLine) int {
	return line.Price * line.Count
}
```

`any` 适合边界层和通用工具，不适合把核心业务模型变成动态类型。
