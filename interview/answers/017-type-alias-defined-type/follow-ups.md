# 017. 类型别名和新类型 - 面试追问

## 1. `type A = B` 和 `type A B` 在赋值上有什么区别？

`type A = B` 是别名，`A` 和 `B` 是同一个类型。

```go
type MyInt = int

func UseInt(n int) {}

var x MyInt = 10
UseInt(x) // 可以
```

`type A B` 是新类型，`A` 和 `B` 是不同类型。

```go
type MyInt int

func UseInt(n int) {}

var x MyInt = 10
// UseInt(x) // 编译错误
UseInt(int(x))
```

面试里可以一句话总结：别名解决兼容问题，新类型解决语义和约束问题。

## 2. 新类型会不会继承底层类型的方法？

不会。

```go
type MyDuration time.Duration

var d MyDuration = MyDuration(time.Second)

// d.String() // 编译错误
```

需要显式转换：

```go
fmt.Println(time.Duration(d).String())
```

或者给新类型定义自己的方法：

```go
func (d MyDuration) String() string {
	return time.Duration(d).String()
}
```

这让你可以控制新类型暴露什么行为，而不是自动继承底层类型的全部 API。

## 3. 为什么 `UserID`、`OrderID` 更适合用新类型？

因为它们虽然底层都可能是 `int64`，但业务含义不同。

```go
type UserID int64
type OrderID int64

func LoadUser(id UserID) {}

var oid OrderID = 1
// LoadUser(oid) // 编译错误
```

如果用别名，就挡不住误传：

```go
type UserID = int64
type OrderID = int64

func LoadUser(id UserID) {}

var oid OrderID = 1
LoadUser(oid) // 可以，但语义错
```

新类型把一部分业务错误提前到编译期。

## 4. 类型别名适合什么重构场景？

适合包迁移、拆分模块、改名时保持兼容。

旧包：

```go
package old

type User struct {
	ID int64
}
```

新包：

```go
package user

type User struct {
	ID int64
}
```

过渡期旧包可以写：

```go
package old

import "example.com/app/user"

type User = user.User
```

这样旧调用方可以继续编译，新代码可以逐步迁移到 `user.User`。

但别名是兼容工具，不是长期设计目标。长期保留太多别名，会让代码库里同一概念有多个入口。

## 5. 别名能不能增强类型安全？

不能。别名只是另一个名字。

```go
type Email = string
type Phone = string

func SendEmail(e Email) {}

var p Phone = "123456"
SendEmail(p) // 可以，因为 Email 和 Phone 都是 string
```

如果要增强类型安全，用新类型：

```go
type Email string
type Phone string

func SendEmail(e Email) {}

var p Phone = "123456"
// SendEmail(p) // 编译错误
```

还可以把校验逻辑放到构造函数里：

```go
func NewEmail(s string) (Email, error) {
	if !strings.Contains(s, "@") {
		return "", fmt.Errorf("invalid email")
	}
	return Email(s), nil
}
```

## 6. 新类型对外暴露时还需要补哪些方法或测试？

常见补充：

- `String()`：日志和调试输出。
- `MarshalJSON` / `UnmarshalJSON`：外部 API。
- `Scan` / `Value`：数据库读写。
- `Valid()` 或构造函数：输入校验。

示例：

```go
type Status int

func (s Status) Valid() bool {
	switch s {
	case StatusPending, StatusRunning, StatusDone:
		return true
	default:
		return false
	}
}

func (s Status) String() string {
	if !s.Valid() {
		return fmt.Sprintf("Status(%d)", s)
	}
	return [...]string{"pending", "running", "done"}[s]
}
```

测试要覆盖非法值：

```go
func TestStatusInvalid(t *testing.T) {
	if Status(99).Valid() {
		t.Fatal("expected invalid status")
	}
}
```

新类型的收益来自“语义集中”，不是只换一个更好看的名字。
