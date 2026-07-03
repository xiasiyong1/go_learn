# 067. 结构体嵌入

## 问题

Go 结构体嵌入、字段提升和方法提升应该怎么理解？

## 先给结论

结构体嵌入是组合，不是继承。匿名字段的字段和方法可以被外层类型“提升”访问，但外层类型不会变成内层类型的子类。嵌入能减少样板代码，也可能把内部能力暴露成外部 API，所以要谨慎使用。

## 1. 嵌入字段是什么

普通字段有字段名：

```go
type User struct {
	Name string
}

type Profile struct {
	User User
}
```

嵌入字段省略字段名，字段名默认就是类型名：

```go
type Profile struct {
	User
}
```

访问时可以写完整路径，也可以用提升后的短路径：

```go
p := Profile{
	User: User{Name: "alice"},
}

fmt.Println(p.User.Name) // 完整路径
fmt.Println(p.Name)      // 字段提升
```

`p.Name` 只是语法便利，不表示 `Profile` 继承了 `User`。

## 2. 嵌入不是继承

外层类型不能当成内层类型使用。

```go
type User struct {
	Name string
}

type Admin struct {
	User
	Level int
}

func printUser(u User) {
	fmt.Println(u.Name)
}

func main() {
	a := Admin{User: User{Name: "root"}, Level: 1}

	// printUser(a) // 编译失败：Admin 不是 User
	printUser(a.User)
}
```

如果面试官问“Go 有没有继承”，比较准确的回答是：没有传统继承，Go 鼓励组合；嵌入提供字段和方法提升，但不建立父子类型关系。

## 3. 方法也会被提升

内层类型的方法可以通过外层值调用。

```go
type Logger struct{}

func (Logger) Print(msg string) {
	fmt.Println(msg)
}

type Service struct {
	Logger
}

func main() {
	s := Service{}
	s.Print("hello")        // 方法提升
	s.Logger.Print("hello") // 完整路径
}
```

这会影响接口实现。

```go
type Printer interface {
	Print(string)
}

var _ Printer = Service{}
```

`Service` 自己没有显式声明 `Print`，但通过嵌入 `Logger` 获得了提升方法，所以可以实现 `Printer`。

## 4. 字段或方法冲突时不会自动猜

如果多个嵌入字段都有同名字段，短选择器会变成歧义。

```go
type A struct {
	Name string
}

type B struct {
	Name string
}

type C struct {
	A
	B
}

func main() {
	c := C{A: A{Name: "a"}, B: B{Name: "b"}}

	// fmt.Println(c.Name) // 编译失败：ambiguous selector c.Name
	fmt.Println(c.A.Name)
	fmt.Println(c.B.Name)
}
```

这说明字段提升不是“合并字段”，只是编译器提供的选择器简写。

## 5. 嵌入导出类型会扩大 API 面积

如果你在导出结构体里嵌入一个导出类型，内层导出方法也可能变成外层 API 的一部分。

```go
type Client struct{}

func (Client) Close() error { return nil }
func (Client) Reset()       {}

type Service struct {
	Client // Close 和 Reset 都被提升到 Service 上
}
```

调用方可以这样写：

```go
var s Service
_ = s.Close()
s.Reset()
```

以后你想移除 `Client` 嵌入，就可能破坏调用方。公共 API 里更稳的方式是使用命名字段：

```go
type Service struct {
	client Client
}
```

## 6. 嵌入锁要特别小心复制

嵌入 `sync.Mutex` 很常见，但含锁结构体使用后不应该复制。

```go
type Cache struct {
	sync.Mutex
	m map[string]string
}

func bad(c Cache) {
	c.Lock()
	defer c.Unlock()
}
```

`bad(c Cache)` 会复制锁。更好的写法是传指针：

```go
func good(c *Cache) {
	c.Lock()
	defer c.Unlock()
}
```

这类问题可以用 `go vet -copylocks` 辅助发现。

## 7. 面试时怎么答

可以这样回答：

- 嵌入是组合语法，不是继承。
- 匿名字段的字段和方法可以被提升，外层可以用短选择器访问。
- 提升不改变类型关系，外层类型不能直接当内层类型传参。
- 多个嵌入字段有同名成员时会歧义，必须显式写路径。
- 公共结构体嵌入导出类型会扩大 API，要谨慎。
- 嵌入含锁类型时要避免复制外层结构体。

## 面试追问

追问参考答案：[follow-ups.md](follow-ups.md)

- 为什么说嵌入不是继承？
- 方法提升会不会让外层类型实现接口？
- 字段名冲突时 Go 怎么处理？
- 什么时候应该用嵌入，什么时候应该用普通字段？
- 为什么嵌入 `sync.Mutex` 后不能随便复制结构体？
