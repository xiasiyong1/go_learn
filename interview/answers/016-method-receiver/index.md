# 016. 值接收者和指针接收者

## 问题

值接收者和指针接收者应该怎么选？

## 先给结论

值接收者会复制接收者；指针接收者共享原对象。

选择接收者时主要看四件事：

- 方法是否需要修改接收者。
- 接收者复制成本是否明显。
- 类型是否包含锁、buffer、缓存、连接等不能随意复制的状态。
- 这个类型要如何实现接口。

一个类型的方法接收者最好保持一致。只要某些方法必须用指针接收者，通常这个类型的其他方法也倾向使用指针接收者。

## 值接收者修改的是副本

```go
type Counter struct {
	n int
}

func (c Counter) Inc() {
	c.n++
}

func main() {
	c := Counter{}
	c.Inc()
	fmt.Println(c.n) // 0
}
```

`Inc` 里的 `c` 是副本，修改不会影响原对象。

要修改原对象，用指针接收者：

```go
func (c *Counter) Inc() {
	c.n++
}

func main() {
	c := Counter{}
	c.Inc()
	fmt.Println(c.n) // 1
}
```

这里 `c.Inc()` 能调用指针接收者方法，是因为变量 `c` 可寻址，编译器帮你做了 `(&c).Inc()`。

## 方法接收者影响接口实现

方法集规则：

- `T` 的方法集只包含值接收者方法。
- `*T` 的方法集包含值接收者方法和指针接收者方法。

```go
type Writer interface {
	Write([]byte) (int, error)
}

type Buffer struct {
	data []byte
}

func (b *Buffer) Write(p []byte) (int, error) {
	b.data = append(b.data, p...)
	return len(p), nil
}

var _ Writer = (*Buffer)(nil) // 可以
// var _ Writer = Buffer{}    // 编译错误
```

因为 `Write` 是指针接收者方法，所以只有 `*Buffer` 实现了 `Writer`。

这也是依赖注入和 mock 里常见的编译问题：你传了 `T`，但接口需要的是 `*T` 的方法集。

## 自动取地址只是调用语法糖

```go
var b Buffer
b.Write([]byte("go")) // 编译器可改写成 (&b).Write(...)
```

但这不代表 `Buffer` 实现了 `Writer`：

```go
func Use(w Writer) {}

var b Buffer
// Use(b)  // 编译错误
Use(&b)    // 正确
```

自动取地址只发生在具体方法调用语法上，不改变接口方法集规则。

## 什么时候用值接收者

适合小而不可变的值类型：

```go
type Point struct {
	X int
	Y int
}

func (p Point) Move(dx, dy int) Point {
	p.X += dx
	p.Y += dy
	return p
}
```

`Move` 返回新值，不修改原值，值语义很清楚。

类似 `time.Time` 这种类型也常用值语义：一个时间点就是一个值，方法不会改变原对象。

## 什么时候用指针接收者

需要修改接收者：

```go
func (c *Counter) Inc() {
	c.n++
}
```

避免复制大对象：

```go
type Big struct {
	data [4096]byte
}

func (b *Big) Reset() {
	for i := range b.data {
		b.data[i] = 0
	}
}
```

包含锁或内部状态：

```go
type SafeCounter struct {
	mu sync.Mutex
	n  int
}

func (c *SafeCounter) Inc() {
	c.mu.Lock()
	defer c.mu.Unlock()
	c.n++
}
```

这种类型如果用值接收者，会复制锁和状态，通常是 bug。

## 不要随便混用接收者

```go
type User struct {
	Name string
}

func (u User) NameLen() int {
	return len(u.Name)
}

func (u *User) Rename(name string) {
	u.Name = name
}
```

这种混用有时可以解释：`NameLen` 不修改，`Rename` 修改。但公共类型里大量混用会增加方法集和接口实现的理解成本。

更稳的经验是：

- 类型有任何指针接收者方法时，优先统一用指针接收者。
- 小型不可变值类型，统一用值接收者。
- 不要为了“这个方法不修改”就在一个明显可变类型里改用值接收者。

## 方法值也受接收者影响

值接收者方法值会复制接收者：

```go
type Counter struct {
	n int
}

func (c Counter) Value() int {
	return c.n
}

c := Counter{n: 1}
value := c.Value
c.n = 2

fmt.Println(value()) // 1
```

指针接收者方法值绑定指针：

```go
func (c *Counter) PtrValue() int {
	return c.n
}

c := Counter{n: 1}
value := c.PtrValue
c.n = 2

fmt.Println(value()) // 2
```

这类细节在回调和延迟执行里很容易变成 bug。

## 面试追问

追问参考答案：[follow-ups.md](follow-ups.md)

- 值接收者和指针接收者在修改字段时有什么区别？
- 方法集规则如何影响接口实现？
- 为什么 `b.Write()` 能调用，但 `Use(b)` 不能通过接口检查？
- 含锁结构体为什么应该用指针接收者？
- 方法值绑定接收者时会发生什么？
- 同一个类型能不能混用值接收者和指针接收者？
