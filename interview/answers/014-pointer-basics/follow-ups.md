# 014. 指针基础 - 面试追问

## 1. Go 指针和 C 指针最大的区别是什么？

Go 指针可以取地址和解引用，但不能做普通指针运算。

```go
n := 10
p := &n

fmt.Println(*p) // 10

// p++     // 编译错误
// p = p+1 // 编译错误
```

不能做指针运算的好处是：

- 不能随便越界访问内存。
- 不能通过地址运算伪造对象。
- GC 可以准确追踪对象引用。
- 普通业务代码更少接触野指针、悬垂指针问题。

如果需要连续访问数据，用切片和索引：

```go
xs := []int{1, 2, 3}
fmt.Println(xs[1])
```

`unsafe.Pointer` 可以绕过这些限制，但那是显式放弃部分安全性，应该单独说明边界。

## 2. nil 指针作为参数时应该怎么设计 API？

先决定 nil 是否是合法输入。

如果 nil 有业务含义，方法里要明确处理：

```go
func DisplayName(u *User) string {
	if u == nil {
		return "<guest>"
	}
	return u.Name
}
```

如果 nil 不合法，尽早返回错误：

```go
func NewHandler(store *Store) (*Handler, error) {
	if store == nil {
		return nil, fmt.Errorf("store is nil")
	}
	return &Handler{store: store}, nil
}
```

不推荐既不检查，也不说明：

```go
func DisplayName(u *User) string {
	return u.Name // u 为 nil 时 panic
}
```

除非这是内部函数，并且调用约束非常明确，否则这种 API 对初学者和调用方都不友好。

## 3. 为什么 Go 可以返回局部变量地址？

因为编译器会做逃逸分析。局部变量如果在函数返回后仍然被引用，就不会放在会失效的位置。

```go
func NewUser(name string) *User {
	u := User{Name: name}
	return &u
}
```

这在 Go 中是安全的。编译器会保证 `u` 在函数返回后仍然有效，通常会让它逃逸到堆上。

可以用命令看逃逸信息：

```sh
go build -gcflags=-m ./...
```

面试里不要说“返回局部变量地址会悬垂”。这是把其他语言经验错误套到 Go。

## 4. 传指针一定比传值快吗？

不一定。

传指针减少了值复制，但可能让对象逃逸到堆上，也会引入共享可变状态。

```go
type Point struct {
	X, Y int
}

func Distance(p Point) int {
	return p.X*p.X + p.Y*p.Y
}
```

这种小结构体传值通常很自然。

较大的对象或需要修改原对象时才更适合指针：

```go
type Buffer struct {
	data [4096]byte
	n    int
}

func Reset(b *Buffer) {
	b.n = 0
}
```

性能判断要靠数据：

```sh
go test -bench . -benchmem
go build -gcflags=-m ./...
```

更稳的回答是：先按语义选择，性能敏感时再验证。

## 5. 值拷贝里有 slice/map 字段时，底层数据会不会共享？

会共享。

```go
type Config struct {
	Tags []string
	Meta map[string]string
}

a := Config{
	Tags: []string{"go"},
	Meta: map[string]string{"env": "dev"},
}
b := a

b.Tags[0] = "java"
b.Meta["env"] = "prod"

fmt.Println(a.Tags[0])    // java
fmt.Println(a.Meta["env"]) // prod
```

结构体是值拷贝，但 slice 和 map 字段本身是描述符，描述符指向的底层数据仍然是同一份。

需要深拷贝时要手动复制：

```go
func CloneConfig(c Config) Config {
	tags := append([]string(nil), c.Tags...)

	meta := make(map[string]string, len(c.Meta))
	for k, v := range c.Meta {
		meta[k] = v
	}

	return Config{Tags: tags, Meta: meta}
}
```

## 6. 为什么含 `sync.Mutex` 的结构体使用后不能复制？

因为复制后会得到另一个锁状态和另一份字段，锁保护关系被破坏。

```go
type Counter struct {
	mu sync.Mutex
	n  int
}

func Bad(c Counter) {
	c.mu.Lock()
	defer c.mu.Unlock()
	c.n++
}
```

`Bad` 复制了 `Counter`，它锁住的是副本，不是原对象。

正确写法：

```go
func Good(c *Counter) {
	c.mu.Lock()
	defer c.mu.Unlock()
	c.n++
}
```

方法接收者也一样，含锁类型通常用指针接收者：

```go
func (c *Counter) Inc() {
	c.mu.Lock()
	defer c.mu.Unlock()
	c.n++
}
```

可以用 `go vet` 辅助检查：

```sh
go vet -copylocks ./...
```
