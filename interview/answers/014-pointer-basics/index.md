# 014. 指针基础

## 问题

Go 指针能做什么？为什么不能做指针运算？

## 先给结论

Go 指针保存变量地址，可以通过指针间接访问和修改同一个对象。

Go 不支持普通指针运算，比如 `p++` 或 `p + 1`。这样做是为了保持内存安全，也让 GC 能准确追踪对象引用关系。

面试时不要简单说“传指针更快”。更好的回答是：指针表达共享和可变，值表达复制和独立；性能要结合对象大小、逃逸、缓存局部性和 API 语义判断。

## `&` 取地址，`*` 解引用

```go
func main() {
	n := 10
	p := &n

	fmt.Println(p)  // 地址
	fmt.Println(*p) // 10

	*p = 20
	fmt.Println(n) // 20
}
```

`p` 指向 `n`，通过 `*p` 修改的就是 `n` 本身。

函数里传指针，可以修改调用方对象：

```go
func AddOne(n *int) {
	(*n)++
}

func main() {
	x := 1
	AddOne(&x)
	fmt.Println(x) // 2
}
```

这里写成 `(*n)++` 是为了让“先解引用，再自增”的意图更清楚。

## nil 指针解引用会 panic

```go
var p *int
fmt.Println(p == nil) // true

fmt.Println(*p) // panic
```

如果指针参数允许为 nil，要显式处理：

```go
func Name(u *User) string {
	if u == nil {
		return "<unknown>"
	}
	return u.Name
}
```

如果 nil 不是合法输入，通常尽早返回错误或让构造函数保证非 nil。

```go
func NewService(repo *Repo) (*Service, error) {
	if repo == nil {
		return nil, fmt.Errorf("repo is nil")
	}
	return &Service{repo: repo}, nil
}
```

## 为什么不能做指针运算

C 里常见的写法：

```c
p++;
```

Go 的普通指针不允许这么做：

```go
// p++     // 编译错误
// p = p+1 // 编译错误
```

原因包括：

- 防止越界访问任意内存。
- 避免伪造地址破坏类型安全。
- 让 GC 能准确知道哪些值是指针、指向哪些对象。
- 减少悬垂指针、野指针这类问题。

Go 允许通过数组、切片和索引表达连续内存访问：

```go
xs := []int{10, 20, 30}
fmt.Println(xs[1]) // 20
```

如果确实要做底层内存操作，需要 `unsafe`，但那已经是另一个高阶话题，不属于普通指针基础。

## 可以返回局部变量指针

这在 Go 里是安全的：

```go
func NewInt() *int {
	n := 10
	return &n
}
```

编译器会做逃逸分析。如果 `n` 的地址返回给函数外部，`n` 会被放到函数返回后仍然有效的位置，通常是堆上。

可以用命令观察：

```sh
go build -gcflags=-m ./...
```

所以不要把 C/C++ 里“不能返回局部变量地址”的经验直接套到 Go 上。

## 传值和传指针怎么选

适合传指针：

- 函数需要修改调用方对象。
- 对象较大，复制成本明显。
- 类型内部有锁、文件、连接等不能随意复制的字段。
- 方法需要保持一致的方法集，尤其是有指针接收者时。

```go
type Counter struct {
	mu sync.Mutex
	n  int
}

func (c *Counter) Add() {
	c.mu.Lock()
	defer c.mu.Unlock()
	c.n++
}
```

适合传值：

- 小对象。
- 不希望函数修改原对象。
- 值本身表达不可变语义，比如 `time.Time`、坐标、配置快照。

```go
type Point struct {
	X, Y int
}

func Move(p Point, dx, dy int) Point {
	p.X += dx
	p.Y += dy
	return p
}
```

## 值拷贝不一定是深拷贝

如果结构体里有 slice、map、指针字段，拷贝结构体只会复制这些字段的描述符或地址，底层数据仍然共享。

```go
type Bag struct {
	Items []string
}

func main() {
	a := Bag{Items: []string{"go"}}
	b := a

	b.Items[0] = "java"

	fmt.Println(a.Items[0]) // java
}
```

如果需要真正独立，要复制底层数据：

```go
func CloneBag(b Bag) Bag {
	items := make([]string, len(b.Items))
	copy(items, b.Items)
	return Bag{Items: items}
}
```

这个追问很常见，因为很多人把“值传递”误以为“所有内部数据都独立”。

## 传指针不一定更快

传指针少复制了一份值，但可能带来：

- 对象逃逸到堆上。
- GC 需要扫描更多指针。
- 共享可变状态导致并发风险。
- 缓存局部性变差。
- API 暗示调用方对象可能被修改。

小结构体直接传值通常更清晰：

```go
type Range struct {
	Start int
	End   int
}

func Contains(r Range, n int) bool {
	return n >= r.Start && n < r.End
}
```

是否需要优化，要用 benchmark 和逃逸分析验证：

```sh
go test -bench . -benchmem
go build -gcflags=-m ./...
```

## 含锁结构体不要复制

```go
type SafeCounter struct {
	mu sync.Mutex
	n  int
}

func Bad(c SafeCounter) {
	c.mu.Lock()
	defer c.mu.Unlock()
	c.n++
}
```

这里复制了 `SafeCounter`，锁和数据都被复制，保护的已经不是原对象。

应该传指针：

```go
func Good(c *SafeCounter) {
	c.mu.Lock()
	defer c.mu.Unlock()
	c.n++
}
```

可以用工具检查：

```sh
go vet -copylocks ./...
```

## 面试追问

追问参考答案：[follow-ups.md](follow-ups.md)

- Go 指针和 C 指针最大的区别是什么？
- nil 指针作为参数时应该怎么设计 API？
- 为什么 Go 可以返回局部变量地址？
- 传指针一定比传值快吗？
- 值拷贝里有 slice/map 字段时，底层数据会不会共享？
- 为什么含 `sync.Mutex` 的结构体使用后不能复制？
