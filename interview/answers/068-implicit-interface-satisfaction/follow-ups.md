# 068. 接口 - 面试追问

## 1. 为什么 Go 不需要 `implements` 关键字？

因为 Go 用结构化类型关系判断接口实现：只看方法集是否匹配，不看类型是否显式声明。

```go
type Doer interface {
	Do() error
}

type Task struct{}

func (Task) Do() error {
	return nil
}

func run(d Doer) error {
	return d.Do()
}

func main() {
	run(Task{})
}
```

这种方式让 `Task` 不需要知道 `Doer` 的存在。后续另一个包定义了同样方法需求的接口，`Task` 也能自然满足。

代价是：接口实现关系不一定一眼可见，所以关键位置可以用编译期断言补充说明。

## 2. 为什么说接口应该由使用方定义？

使用方知道自己需要什么。生产者往往容易定义过大的接口。

不好的生产者式接口：

```go
type UserRepository interface {
	Create(User) error
	Update(User) error
	Delete(string) error
	Find(string) (User, error)
	List() ([]User, error)
}
```

如果某个服务只需要查询用户，就定义最小接口：

```go
type UserFinder interface {
	Find(string) (User, error)
}

type ProfileService struct {
	users UserFinder
}
```

测试时 fake 实现也更简单：

```go
type fakeUsers struct{}

func (fakeUsers) Find(id string) (User, error) {
	return User{ID: id}, nil
}
```

接口越小，替换越容易，测试越轻。

## 3. 值接收者和指针接收者如何影响接口实现？

值接收者方法属于 `T` 和 `*T`：

```go
type Namer interface {
	Name() string
}

type User struct{ name string }

func (u User) Name() string { return u.name }

var _ Namer = User{}
var _ Namer = (*User)(nil)
```

指针接收者方法只属于 `*T`：

```go
type Resetter interface {
	Reset()
}

type Buffer struct{}

func (b *Buffer) Reset() {}

// var _ Resetter = Buffer{}    // 编译失败
var _ Resetter = (*Buffer)(nil) // 正确
```

为什么？因为指针接收者方法可能要修改原对象。接口里如果存的是不可寻址的值，编译器不能保证能自动取地址。

## 4. 空接口和小接口有什么区别？

空接口 `any` 表示“不知道对方有什么行为”，只能保存任意值。

```go
func printAny(v any) {
	fmt.Printf("%T %v\n", v, v)
}
```

小接口表示“我只需要这一点行为”。

```go
type Reader interface {
	Read([]byte) (int, error)
}

func readAll(r Reader) ([]byte, error) {
	return io.ReadAll(r)
}
```

`any` 会丢失编译期行为约束，后续通常需要类型断言。

```go
func length(v any) int {
	s, ok := v.(string)
	if !ok {
		return 0
	}
	return len(s)
}
```

所以业务抽象优先用小接口，而不是 `any`。

## 5. 编译期接口断言应该写在哪里？

写在具体类型所在包或相关测试文件里都可以，取决于这个约束是不是类型设计的一部分。

公共扩展点建议和类型放在一起：

```go
type Plugin struct{}

func (p *Plugin) Start(context.Context) error {
	return nil
}

var _ Starter = (*Plugin)(nil)
```

如果只是测试期验证 fake 实现接口，可以放测试文件：

```go
type fakeSender struct{}

func (fakeSender) Send(string) error {
	return nil
}

var _ Sender = fakeSender{}
```

断言本身不会生成业务逻辑，它只是让编译器帮你在接口不匹配时立刻报错。
