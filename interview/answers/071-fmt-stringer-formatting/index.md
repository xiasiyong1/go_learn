# 071. 格式化

## 问题

`fmt` 常用格式化占位符和 `Stringer` 接口应该怎么用？

## 先给结论

`fmt` 的核心不是背所有占位符，而是知道调试时怎么观察类型和值，输出给用户时怎么控制字符串表示。`fmt.Stringer` 可以自定义类型的文本表现，但 `String()` 方法要避免递归、避免副作用、避免泄露敏感信息。

## 1. 常用占位符怎么选

观察值：

```go
u := User{Name: "alice", Age: 18}

fmt.Printf("%v\n", u)  // {alice 18}
fmt.Printf("%+v\n", u) // {Name:alice Age:18}
fmt.Printf("%#v\n", u) // main.User{Name:"alice", Age:18}
fmt.Printf("%T\n", u)  // main.User
```

调试字符串和字节：

```go
s := "go\n"
b := []byte(s)

fmt.Printf("%q\n", s) // "go\n"
fmt.Printf("%x\n", b) // 676f0a
```

面试时可以说：`%+v` 适合看结构体字段名，`%#v` 适合看 Go 语法风格表示，`%T` 适合看动态类型。

## 2. `Stringer` 控制默认字符串表示

实现 `String() string` 后，`fmt` 会在很多场景调用它。

```go
type Status int

const (
	StatusPending Status = iota
	StatusDone
)

func (s Status) String() string {
	switch s {
	case StatusPending:
		return "pending"
	case StatusDone:
		return "done"
	default:
		return fmt.Sprintf("Status(%d)", int(s))
	}
}

fmt.Println(StatusDone) // done
```

枚举、状态、ID 包装类型很适合实现 `Stringer`，因为它能把内部值转成更可读的文案。

## 3. `String()` 方法里不要递归调用自己

错误示例：

```go
type User struct {
	Name string
}

func (u User) String() string {
	return fmt.Sprintf("%v", u) // 会再次调用 User.String，递归
}
```

正确写法是格式化具体字段，或者转换成别名类型避免再次触发 `String`。

```go
func (u User) String() string {
	return fmt.Sprintf("User{Name:%q}", u.Name)
}
```

## 4. `%w` 只用于错误包装

`%w` 不是普通打印占位符，它只在 `fmt.Errorf` 中有错误包装语义。

```go
err := os.ErrNotExist
wrapped := fmt.Errorf("load config: %w", err)

fmt.Println(errors.Is(wrapped, os.ErrNotExist)) // true
```

如果写 `%v`，错误字符串看起来类似，但错误链断了：

```go
wrapped := fmt.Errorf("load config: %v", err)
fmt.Println(errors.Is(wrapped, os.ErrNotExist)) // false
```

面试里可以把它和 error wrapping 联系起来。

## 5. 不要把 fmt 当成高性能序列化工具

`fmt` 很适合调试和普通文本输出，但它走通用格式化逻辑，性能和分配通常不如专门方法。

```go
// 调试可以
fmt.Sprintf("id=%d name=%s", id, name)
```

热点路径构造字符串可以用 `strings.Builder`：

```go
var b strings.Builder
b.WriteString("id=")
b.WriteString(strconv.Itoa(id))
b.WriteString(" name=")
b.WriteString(name)
out := b.String()
```

结构化数据用 JSON、protobuf 或专门编码，不要用 fmt 拼协议。

## 6. String 输出要避免泄露敏感信息

```go
type Token struct {
	Value string
}

func (t Token) String() string {
	return "Token(***)"
}
```

如果直接输出真实 token，日志和错误信息都可能泄露敏感数据。

## 7. 面试时怎么答

可以这样回答：

- `%v` 看值，`%+v` 看结构体字段名，`%#v` 看 Go 语法风格，`%T` 看类型。
- `%q` 适合看带转义的字符串，`%x` 适合看字节。
- 实现 `String() string` 可以控制类型的默认文本输出。
- `String()` 里不要用 `%v` 格式化自己，否则可能递归。
- `%w` 只用于 `fmt.Errorf` 包装错误。
- 生产日志要注意性能、结构化和敏感信息脱敏。

## 面试追问

追问参考答案：[follow-ups.md](follow-ups.md)

- `%v`、`%+v`、`%#v`、`%T` 分别适合什么场景？
- `Stringer` 为什么会导致递归？
- `%w` 和 `%v` 包装 error 有什么区别？
- 为什么日志里不能随便打印结构体？
- 热点路径里为什么要少用 `fmt.Sprintf`？
