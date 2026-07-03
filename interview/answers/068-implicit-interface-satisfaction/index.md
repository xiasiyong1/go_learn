# 068. 接口

## 问题

Go 的接口为什么是隐式实现？接口应该由谁来定义？

## 先给结论

Go 没有 `implements` 关键字。一个类型只要拥有接口要求的方法集，就自动实现这个接口。接口更适合由“使用方”定义，因为使用方最清楚自己需要哪些行为。好的 Go 接口通常很小，最好只描述调用方真正依赖的方法。

## 1. 隐式实现是什么

类型不需要声明自己实现了哪个接口。

```go
type Reader interface {
	Read([]byte) (int, error)
}

type File struct{}

func (File) Read(p []byte) (int, error) {
	return 0, io.EOF
}

func use(r Reader) {}

func main() {
	var f File
	use(f) // File 自动满足 Reader
}
```

`File` 不需要 import 定义 `Reader` 的包，也不需要写 `implements Reader`。这降低了包之间的耦合。

## 2. 接口由使用方定义

如果一个函数只需要发送邮件，就定义它需要的最小能力。

```go
type MailSender interface {
	Send(to string, body string) error
}

type Service struct {
	sender MailSender
}

func (s Service) Notify(user string) error {
	return s.sender.Send(user, "hello")
}
```

真实实现可以很复杂：

```go
type SMTPClient struct {
	addr string
}

func (c *SMTPClient) Send(to string, body string) error {
	// send by smtp
	return nil
}
```

测试实现可以很简单：

```go
type fakeSender struct {
	called bool
}

func (f *fakeSender) Send(to string, body string) error {
	f.called = true
	return nil
}
```

如果生产者提前定义一个很大的接口，测试方就会被迫实现很多不需要的方法。

## 3. 方法集决定是否实现接口

值接收者方法同时属于 `T` 和 `*T` 的方法集。

```go
type Stringer interface {
	String() string
}

type User struct {
	Name string
}

func (u User) String() string {
	return u.Name
}

var _ Stringer = User{}
var _ Stringer = (*User)(nil)
```

指针接收者方法只属于 `*T` 的方法集。

```go
type Closer interface {
	Close() error
}

type Conn struct{}

func (c *Conn) Close() error {
	return nil
}

// var _ Closer = Conn{}       // 编译失败
var _ Closer = (*Conn)(nil) // 正确
```

这也是很多接口实现问题的根源：你传的是值，但只有指针实现了接口。

## 4. 小接口比大接口更灵活

`io.Reader` 只有一个方法，所以任何能读数据的类型都能适配它。

```go
func countBytes(r io.Reader) (int64, error) {
	return io.Copy(io.Discard, r)
}

countBytes(strings.NewReader("hello"))
```

反过来，大接口会让实现变重。

```go
type BadStorage interface {
	Get(string) ([]byte, error)
	Set(string, []byte) error
	Delete(string) error
	List() ([]string, error)
	Close() error
}
```

如果某个函数只需要 `Get`，就不要依赖整个 `BadStorage`。

```go
type Getter interface {
	Get(string) ([]byte, error)
}
```

## 5. 编译期断言用来记录关键约束

当你希望某个类型必须实现某个接口，可以写断言。

```go
var _ io.Reader = (*bytes.Buffer)(nil)
```

自定义类型也一样：

```go
type Handler interface {
	Handle(context.Context) error
}

type Job struct{}

func (j *Job) Handle(ctx context.Context) error {
	return nil
}

var _ Handler = (*Job)(nil)
```

这不是必须写，但对公共类型、插件类型、框架扩展点很有价值。

## 6. 面试时怎么答

可以这样回答：

- Go 接口是隐式实现，类型只要方法集匹配就实现接口。
- 没有 `implements` 关键字，这让类型和接口包之间更解耦。
- 接口通常由使用方定义，因为使用方知道自己依赖哪些行为。
- 接口越小越容易实现、测试和复用。
- 方法集很关键：指针接收者方法只让 `*T` 实现接口，值接收者方法让 `T` 和 `*T` 都可用。
- 重要接口关系可以用 `var _ Interface = (*T)(nil)` 做编译期断言。

## 面试追问

追问参考答案：[follow-ups.md](follow-ups.md)

- 为什么 Go 不需要 `implements` 关键字？
- 为什么说接口应该由使用方定义？
- 值接收者和指针接收者如何影响接口实现？
- 空接口和小接口有什么区别？
- 编译期接口断言应该写在哪里？
