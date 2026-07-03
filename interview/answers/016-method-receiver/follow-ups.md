# 016. 值接收者和指针接收者 - 面试追问

## 1. 值接收者和指针接收者在修改字段时有什么区别？

值接收者修改副本：

```go
type Counter struct {
	N int
}

func (c Counter) Inc() {
	c.N++
}

c := Counter{}
c.Inc()
fmt.Println(c.N) // 0
```

指针接收者修改原对象：

```go
func (c *Counter) IncPtr() {
	c.N++
}

c := Counter{}
c.IncPtr()
fmt.Println(c.N) // 1
```

所以如果方法语义是“改变这个对象”，就应该用指针接收者。

## 2. 方法集规则如何影响接口实现？

规则是：

- `T` 的方法集包含值接收者方法。
- `*T` 的方法集包含值接收者和指针接收者方法。

```go
type Flusher interface {
	Flush() error
}

type Buffer struct{}

func (b *Buffer) Flush() error {
	return nil
}

var _ Flusher = (*Buffer)(nil) // 正确
// var _ Flusher = Buffer{}    // 编译错误
```

因为 `Flush` 是 `*Buffer` 的方法，不是 `Buffer` 方法集里的方法。

如果 `Flush` 改成值接收者：

```go
func (b Buffer) Flush() error {
	return nil
}
```

那么 `Buffer` 和 `*Buffer` 都实现 `Flusher`。

## 3. 为什么 `b.Write()` 能调用，但 `Use(b)` 不能通过接口检查？

因为可寻址变量调用指针接收者方法时，编译器可以自动取地址：

```go
var b Buffer
b.Write([]byte("go")) // 等价于 (&b).Write(...)
```

但接口赋值检查看的是方法集，不会因为 `b` 可寻址就把 `Buffer` 当成 `*Buffer`。

```go
type Writer interface {
	Write([]byte) (int, error)
}

func Use(w Writer) {}

var b Buffer
// Use(b) // 编译错误
Use(&b)   // 正确
```

一句话：自动取地址是调用语法糖，不改变接口实现规则。

## 4. 含锁结构体为什么应该用指针接收者？

因为值接收者会复制锁。

```go
type SafeCounter struct {
	mu sync.Mutex
	n  int
}

func (c SafeCounter) Inc() {
	c.mu.Lock()
	defer c.mu.Unlock()
	c.n++
}
```

这段代码锁住的是副本，原对象没有被正确保护和修改。

正确写法：

```go
func (c *SafeCounter) Inc() {
	c.mu.Lock()
	defer c.mu.Unlock()
	c.n++
}
```

可以用工具辅助：

```sh
go vet -copylocks ./...
```

## 5. 方法值绑定接收者时会发生什么？

方法值会保存接收者。

值接收者保存副本：

```go
type User struct {
	Name string
}

func (u User) Hello() string {
	return u.Name
}

u := User{Name: "Tom"}
hello := u.Hello

u.Name = "Jerry"

fmt.Println(hello())  // Tom
fmt.Println(u.Hello()) // Jerry
```

指针接收者保存指针：

```go
func (u *User) PtrHello() string {
	return u.Name
}

u := User{Name: "Tom"}
hello := u.PtrHello

u.Name = "Jerry"

fmt.Println(hello()) // Jerry
```

在回调、goroutine、`defer` 中保存方法值时，这个差异很重要。

## 6. 同一个类型能不能混用值接收者和指针接收者？

语法上可以，但要谨慎。

```go
type User struct {
	Name string
}

func (u User) DisplayName() string {
	return u.Name
}

func (u *User) Rename(name string) {
	u.Name = name
}
```

这种小类型可以解释得通：读方法值接收者，写方法指针接收者。

但如果类型明显是可变对象，或者有锁、缓存、连接等状态，建议统一指针接收者：

```go
type Store struct {
	mu sync.Mutex
	m  map[string]string
}

func (s *Store) Get(key string) string { return s.m[key] }
func (s *Store) Set(key, value string) { s.m[key] = value }
```

统一接收者能减少方法集意外，也让接口实现更容易预测。
