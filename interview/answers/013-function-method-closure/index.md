# 013. 函数、方法、闭包和可变参数

## 问题

Go 中函数、方法、闭包和可变参数分别有什么特点？

## 先给结论

- 函数是值，可以赋值、传参、返回。
- 方法是带接收者的函数，用来把行为绑定到类型上。
- 闭包会捕获外部变量，可能延长变量生命周期。
- 可变参数在函数内部表现为切片，调用时已有切片要用 `...` 展开。

这题看起来基础，但面试追问通常会落到方法值、方法表达式、闭包捕获变量、循环 goroutine 和可变参数的切片语义。

## 函数是值

函数可以赋值给变量：

```go
func add(a, b int) int {
	return a + b
}

func main() {
	var op func(int, int) int
	op = add

	fmt.Println(op(1, 2)) // 3
}
```

函数也可以作为参数：

```go
func Filter(nums []int, keep func(int) bool) []int {
	out := make([]int, 0, len(nums))
	for _, n := range nums {
		if keep(n) {
			out = append(out, n)
		}
	}
	return out
}

evens := Filter([]int{1, 2, 3, 4}, func(n int) bool {
	return n%2 == 0
})
```

这种写法适合表达小的策略、回调、过滤条件。

## 方法是带接收者的函数

```go
type Counter struct {
	n int
}

func (c *Counter) Add(delta int) {
	c.n += delta
}

func (c Counter) Value() int {
	return c.n
}
```

方法接收者本质上可以理解成一个特殊参数。`c.Add(1)` 可以近似理解成把 `c` 作为第一个参数传进去。

方法的意义主要是：

- 把行为组织到类型上。
- 让类型实现接口。
- 表达这个函数和某个数据类型有强关系。

## 方法值和方法表达式

方法值会绑定接收者：

```go
type Counter struct {
	n int
}

func (c *Counter) Add(delta int) {
	c.n += delta
}

func main() {
	c := &Counter{}
	add := c.Add

	add(2)
	add(3)

	fmt.Println(c.n) // 5
}
```

`add := c.Add` 之后，`add` 已经记住了接收者 `c`。

方法表达式不绑定接收者，需要调用时显式传入：

```go
add := (*Counter).Add

c := &Counter{}
add(c, 5)

fmt.Println(c.n) // 5
```

面试里如果能区分这两个概念，说明不是只会写方法调用语法。

## 值接收者的方法值会复制接收者

```go
type Counter struct {
	n int
}

func (c Counter) Value() int {
	return c.n
}

func main() {
	c := Counter{n: 1}
	value := c.Value

	c.n = 2

	fmt.Println(value()) // 1
	fmt.Println(c.Value()) // 2
}
```

`value := c.Value` 会把当时的值接收者复制进方法值里，所以后面修改 `c.n` 不影响 `value()`。

如果是指针接收者，方法值绑定的是指针：

```go
func (c *Counter) PtrValue() int {
	return c.n
}

c := Counter{n: 1}
value := c.PtrValue
c.n = 2

fmt.Println(value()) // 2
```

这个差异常常和“值接收者/指针接收者怎么选”一起追问。

## 闭包捕获变量

闭包可以引用外层函数的变量：

```go
func NewCounter() func() int {
	n := 0
	return func() int {
		n++
		return n
	}
}

func main() {
	next := NewCounter()
	fmt.Println(next()) // 1
	fmt.Println(next()) // 2
}
```

`NewCounter` 返回后，`n` 仍然被闭包引用，所以它的生命周期被延长了。编译器可能让它逃逸到堆上。

可以用命令观察：

```sh
go build -gcflags=-m ./...
```

## 循环和闭包

循环里使用闭包要注意捕获的到底是谁。

更稳的写法是显式把每次迭代的值传给闭包：

```go
for _, name := range []string{"a", "b", "c"} {
	name := name
	go func() {
		fmt.Println(name)
	}()
}
```

或者作为参数传入：

```go
for _, name := range []string{"a", "b", "c"} {
	go func(v string) {
		fmt.Println(v)
	}(name)
}
```

这样每个 goroutine 都拿到自己的值，代码不依赖读者记忆具体 Go 版本的 range 变量语义变化，也更适合初学者理解。

## 可变参数本质上是切片

```go
func Sum(nums ...int) int {
	total := 0
	for _, n := range nums {
		total += n
	}
	return total
}

fmt.Println(Sum(1, 2, 3)) // 6
```

函数内部 `nums` 的类型是 `[]int`。

已有切片调用可变参数函数，要用 `...` 展开：

```go
nums := []int{1, 2, 3}

fmt.Println(Sum(nums...)) // 6
// fmt.Println(Sum(nums)) // 编译错误
```

如果函数修改可变参数切片里的元素，调用方传入切片时会被影响：

```go
func Fill(nums ...int) {
	for i := range nums {
		nums[i] = 0
	}
}

xs := []int{1, 2, 3}
Fill(xs...)
fmt.Println(xs) // [0 0 0]
```

所以可变参数适合只读或明确说明会修改的场景。

## 可变参数适合什么 API

适合少量同类型参数：

```go
func Join(parts ...string) string {
	return strings.Join(parts, "")
}
```

不适合承载复杂配置：

```go
func NewClient(args ...any) *Client {
	// 不推荐：调用方不知道每个参数是什么意思
	return &Client{}
}
```

复杂配置更适合结构体或 functional options：

```go
type ClientOptions struct {
	Timeout time.Duration
	Retries int
}

func NewClient(opt ClientOptions) *Client {
	return &Client{timeout: opt.Timeout, retries: opt.Retries}
}
```

## 面试追问

追问参考答案：[follow-ups.md](follow-ups.md)

- 函数值可以做什么？nil 函数调用会怎样？
- 方法值和方法表达式有什么区别？
- 值接收者的方法值为什么可能看到旧数据？
- 闭包捕获变量会带来什么生命周期和并发问题？
- 循环里启动 goroutine 时，为什么建议显式传参或重新绑定变量？
- 可变参数和切片之间是什么关系？
