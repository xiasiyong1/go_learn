# 017. 类型别名和新类型

## 问题

类型别名和新类型有什么区别？

## 先给结论

类型别名：

```go
type A = B
```

表示 `A` 就是 `B` 的另一个名字。它们是同一个类型。

新定义类型：

```go
type A B
```

表示创建一个新的类型 `A`，它的底层类型是 `B`，但 `A` 和 `B` 是不同类型。

别名主要用于迁移兼容；新类型主要用于表达业务语义和增加类型安全。

## 类型别名就是同一个类型

```go
type MyString = string

func Print(s string) {
	fmt.Println(s)
}

var name MyString = "go"
Print(name) // 可以
```

`MyString` 是 `string` 的别名，在编译器看来它们是同一个类型。

别名也保留原类型的方法集。

```go
type DurationAlias = time.Duration

var d DurationAlias = time.Second
fmt.Println(d.String()) // 1s
```

## 新类型有独立身份

```go
type UserID int64

func LoadUser(id UserID) {}

var n int64 = 100
// LoadUser(n) // 编译错误
LoadUser(UserID(n))
```

`UserID` 的底层类型是 `int64`，但它不是 `int64`。这能避免把普通数字误传给需要用户 ID 的函数。

再比如：

```go
type UserID int64
type OrderID int64

func LoadUser(id UserID) {}

var oid OrderID = 10
// LoadUser(oid) // 编译错误
```

虽然 `UserID` 和 `OrderID` 底层都是 `int64`，但它们代表不同业务概念。

## 新类型不会自动继承方法

```go
type MyDuration time.Duration

var d MyDuration = MyDuration(time.Second)

// fmt.Println(d.String()) // 编译错误
```

`MyDuration` 是新类型，不会自动拥有 `time.Duration` 的方法。

如果需要，可以自己定义方法：

```go
func (d MyDuration) String() string {
	return time.Duration(d).String()
}
```

这也是新类型的价值：你可以决定暴露哪些行为，而不是无条件继承底层类型的方法。

## 新类型可以实现接口

```go
type Status int

const (
	StatusPending Status = iota
	StatusRunning
	StatusDone
)

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

有了 `String()`，`Status` 就实现了 `fmt.Stringer`。

```go
var _ fmt.Stringer = StatusPending
```

这比到处传 `int` 更清楚，也让日志和调试输出更友好。

## 别名适合迁移兼容

假设原来有包：

```go
package olduser

type User struct {
	ID int64
}
```

后来移动到新包：

```go
package user

type User struct {
	ID int64
}
```

为了让旧调用方逐步迁移，可以在旧包里写别名：

```go
package olduser

import "example.com/app/user"

type User = user.User
```

这样旧代码里的 `olduser.User` 和新代码里的 `user.User` 是同一个类型，迁移成本低。

但别名不应该长期滥用。迁移期结束后，最好收敛到新路径，否则代码库里新旧包名长期并存，会增加维护成本。

## 别名不能提供业务约束

```go
type UserID = int64
type OrderID = int64

func LoadUser(id UserID) {}

var oid OrderID = 10
LoadUser(oid) // 可以，因为二者都是 int64
```

如果你想防止 ID 混用，应该用新类型：

```go
type UserID int64
type OrderID int64
```

这时 `LoadUser(oid)` 就会编译失败。

## 新类型对外暴露时要补齐能力

新类型增强了类型安全，但也需要配套能力。

例如 ID 类型常常需要 JSON 编解码、SQL 扫描、字符串化：

```go
type UserID int64

func (id UserID) String() string {
	return strconv.FormatInt(int64(id), 10)
}
```

如果对外 API 使用字符串 ID，还可以实现 JSON：

```go
func (id UserID) MarshalJSON() ([]byte, error) {
	return json.Marshal(id.String())
}
```

面试里可以说：新类型不是只为了“换个名字”，而是为了把业务语义、校验和边界行为聚合起来。

## 面试追问

追问参考答案：[follow-ups.md](follow-ups.md)

- `type A = B` 和 `type A B` 在赋值上有什么区别？
- 新类型会不会继承底层类型的方法？
- 为什么 `UserID`、`OrderID` 更适合用新类型？
- 类型别名适合什么重构场景？
- 别名能不能增强类型安全？
- 新类型对外暴露时还需要补哪些方法或测试？
