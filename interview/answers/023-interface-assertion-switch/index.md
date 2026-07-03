# 023. 类型断言和 type switch

## 问题

空接口、类型断言和 type switch 分别适合什么场景？

## 先给结论

`any` 是 `interface{}` 的别名，表示可以接收任意类型的值。

接口值包含动态类型和动态值。类型断言用于把接口值里的动态值取回成某个具体类型或接口类型；type switch 用于根据动态类型分支处理。

核心取舍：

- 业务主流程尽量保留静态类型。
- 确实面对未知类型时再用 `any`。
- 普通输入分支用双返回值断言，不要用 panic 控制流程。
- type switch 适合有限类型集合；分支太多时要考虑接口或泛型。

## any 适合什么

`any` 可以接收任意值：

```go
func Print(v any) {
	fmt.Printf("%T %v\n", v, v)
}

Print("go")
Print(123)
```

适合：

- 日志字段。
- JSON 任意结构。
- 插件系统边界。
- 调试工具。
- 泛型出现前的一些通用容器。

不适合把业务核心逻辑都写成 `any`：

```go
func BadCreateUser(input any) error {
	m := input.(map[string]any)
	name := m["name"].(string)
	_ = name
	return nil
}
```

这会把类型错误从编译期推迟到运行时。

更好的业务 API 是明确类型：

```go
type CreateUserRequest struct {
	Name string
}

func CreateUser(req CreateUserRequest) error {
	if req.Name == "" {
		return fmt.Errorf("name is required")
	}
	return nil
}
```

## 类型断言

单返回值形式失败会 panic：

```go
var v any = "go"
n := v.(int) // panic
_ = n
```

双返回值形式更适合普通控制流：

```go
var v any = "go"

n, ok := v.(int)
if !ok {
	return fmt.Errorf("expected int, got %T", v)
}
fmt.Println(n)
```

断言目标也可以是接口：

```go
type Stringer interface {
	String() string
}

func Format(v any) string {
	if s, ok := v.(Stringer); ok {
		return s.String()
	}
	return fmt.Sprint(v)
}
```

这不是判断动态类型是不是某个具体类型，而是判断动态值是否实现某个接口。

## type switch

type switch 适合有限类型集合：

```go
func Size(v any) int {
	switch x := v.(type) {
	case string:
		return len(x)
	case []byte:
		return len(x)
	case fmt.Stringer:
		return len(x.String())
	default:
		return 0
	}
}
```

每个 case 里的 `x` 类型不同。

如果分支越来越多：

```go
switch x := v.(type) {
case A:
case B:
case C:
case D:
case E:
}
```

要停下来问：是不是应该让这些类型实现同一个接口？

```go
type Sizer interface {
	Size() int
}
```

或者用泛型让编译期保留类型信息。

## typed nil 仍然要小心

类型断言成功，不代表断言出的指针非 nil。

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
fmt.Println(err == nil) // false
```

`ok == true` 只说明动态类型是 `*MyError`，不说明动态值非 nil。

使用断言结果前，如果它是指针，要按业务需要判 nil。

## 接口应该小而靠近使用方

不推荐生产者定义大接口：

```go
type UserRepository interface {
	Create(User) error
	Update(User) error
	Delete(int64) error
	Find(int64) (User, error)
	List() ([]User, error)
}
```

如果某个服务只需要查用户，就定义小接口：

```go
type UserFinder interface {
	Find(id int64) (User, error)
}

type Service struct {
	users UserFinder
}
```

这样 mock 更容易，依赖更小，也不需要到处类型断言。

## 面试追问

追问参考答案：[follow-ups.md](follow-ups.md)

- `any` 和 `interface{}` 是什么关系？
- 单返回值断言和双返回值断言有什么区别？
- 类型断言目标是接口时，检查的是什么？
- type switch 适合什么场景，什么时候说明设计有问题？
- typed nil 断言成功后为什么还可能拿到 nil 指针？
- 为什么业务核心逻辑不应该到处使用 `any`？
