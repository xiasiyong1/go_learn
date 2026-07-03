# 013. 函数、方法、闭包和可变参数 - 面试追问

## 1. 函数值可以做什么？nil 函数调用会怎样？

函数可以像普通值一样赋值、传参、返回。

```go
func Apply(n int, f func(int) int) int {
	return f(n)
}

double := func(n int) int {
	return n * 2
}

fmt.Println(Apply(3, double)) // 6
```

函数类型的零值是 `nil`。调用 nil 函数会 panic。

```go
var f func()

fmt.Println(f == nil) // true
f() // panic
```

所以回调参数如果允许为空，要先判断：

```go
func Run(after func()) {
	// do work
	if after != nil {
		after()
	}
}
```

## 2. 方法值和方法表达式有什么区别？

方法值会绑定接收者：

```go
type User struct {
	Name string
}

func (u User) Hello(prefix string) string {
	return prefix + u.Name
}

u := User{Name: "Tom"}
hello := u.Hello

fmt.Println(hello("hi ")) // hi Tom
```

方法表达式不绑定接收者，调用时要显式传：

```go
hello := User.Hello

u := User{Name: "Tom"}
fmt.Println(hello(u, "hi ")) // hi Tom
```

区别可以记成：

- 方法值：`u.Hello`，接收者已经定了。
- 方法表达式：`User.Hello`，接收者还是参数。

## 3. 值接收者的方法值为什么可能看到旧数据？

因为值接收者会复制接收者。

```go
type Counter struct {
	N int
}

func (c Counter) Value() int {
	return c.N
}

c := Counter{N: 1}
value := c.Value

c.N = 2

fmt.Println(value())   // 1
fmt.Println(c.Value()) // 2
```

`value := c.Value` 时，方法值里保存了一份当时的 `Counter` 副本。

如果改成指针接收者，方法值绑定的是指针：

```go
func (c *Counter) PtrValue() int {
	return c.N
}

c := Counter{N: 1}
value := c.PtrValue
c.N = 2

fmt.Println(value()) // 2
```

这不是谁优谁劣，而是语义不同：值接收者强调复制，指针接收者强调共享和可变。

## 4. 闭包捕获变量会带来什么生命周期和并发问题？

闭包捕获的是变量。只要闭包还活着，被捕获变量就可能继续活着。

```go
func Accumulator() func(int) int {
	sum := 0
	return func(n int) int {
		sum += n
		return sum
	}
}
```

`Accumulator` 返回后，`sum` 仍然被返回的函数使用。

并发场景下，捕获的变量如果被多个 goroutine 修改，需要同步：

```go
count := 0

for i := 0; i < 10; i++ {
	go func() {
		count++ // 数据竞争
	}()
}
```

正确方向是加锁、用 channel 汇总，或者用 atomic。不要因为闭包写起来方便就忽略共享状态。

## 5. 循环里启动 goroutine 时，为什么建议显式传参或重新绑定变量？

这样每次迭代的值边界更清楚，也减少版本语义和读者理解成本。

推荐：

```go
for _, item := range items {
	item := item
	go func() {
		process(item)
	}()
}
```

或者：

```go
for _, item := range items {
	go func(v Item) {
		process(v)
	}(item)
}
```

不要写成捕获外部可变状态：

```go
var current Item
for _, item := range items {
	current = item
	go func() {
		process(current)
	}()
}
```

这里所有 goroutine 共享同一个 `current`，既可能逻辑错，也可能产生数据竞争。

## 6. 可变参数和切片之间是什么关系？

可变参数在函数内部就是切片。

```go
func PrintAll(args ...string) {
	fmt.Printf("%T %v\n", args, args) // []string
}
```

调用时可以传多个参数：

```go
PrintAll("a", "b")
```

已有切片要用 `...` 展开：

```go
xs := []string{"a", "b"}
PrintAll(xs...)
```

如果函数修改参数元素，传入切片的调用方会看到变化：

```go
func Clear(nums ...int) {
	for i := range nums {
		nums[i] = 0
	}
}

xs := []int{1, 2, 3}
Clear(xs...)
fmt.Println(xs) // [0 0 0]
```

所以设计可变参数 API 时，最好默认只读；如果会修改，要在文档和命名里说清楚。
