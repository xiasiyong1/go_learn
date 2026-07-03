# 010. make 和 new - 面试追问

## 1. `new(T)` 的返回值是什么？它是否会初始化 map/channel？

`new(T)` 返回 `*T`，指向一个 `T` 的零值。

```go
p := new(int)
fmt.Printf("%T %d\n", p, *p) // *int 0
```

如果 `T` 是 map 或 channel，零值就是 nil。`new` 不会把它们变成可写 map 或可通信 channel。

```go
pm := new(map[string]int)
pc := new(chan int)

fmt.Println(*pm == nil) // true
fmt.Println(*pc == nil) // true
```

map 和 channel 要用 `make` 初始化运行时结构：

```go
m := make(map[string]int)
ch := make(chan int)
```

## 2. 为什么 `new(map[string]int)` 后写入仍然 panic？

因为你拿到的是“指向 nil map 的指针”，不是“指向已初始化 map 的指针”。

```go
p := new(map[string]int)
(*p)["go"] = 1 // panic
```

等价于：

```go
var m map[string]int
p := &m
(*p)["go"] = 1 // 仍然是 nil map 写入
```

正确写法：

```go
m := make(map[string]int)
m["go"] = 1
```

面试里可以补一句：map 本身已经是一个描述符，很多场景不需要再取 `*map`。

## 3. `make([]int, 3)` 和 `make([]int, 0, 3)` 的区别是什么？

`make([]int, 3)` 表示切片长度是 3，已经有 3 个元素。

```go
s := make([]int, 3)
fmt.Println(s)      // [0 0 0]
fmt.Println(len(s)) // 3

s = append(s, 9)
fmt.Println(s) // [0 0 0 9]
```

`make([]int, 0, 3)` 表示当前长度是 0，只是预留容量 3。

```go
s := make([]int, 0, 3)
fmt.Println(s)      // []
fmt.Println(len(s)) // 0

s = append(s, 9)
fmt.Println(s) // [9]
```

如果后面用下标赋值，用 `len=n`：

```go
out := make([]int, len(in))
for i, v := range in {
	out[i] = v * 2
}
```

如果后面用 append 收集，用 `len=0, cap=n`：

```go
out := make([]int, 0, len(in))
for _, v := range in {
	if v > 0 {
		out = append(out, v)
	}
}
```

## 4. `make(map[K]V, n)` 的 `n` 是长度还是容量？

是容量提示，不是长度，也不是上限。

```go
m := make(map[string]int, 2)
fmt.Println(len(m)) // 0

m["a"] = 1
m["b"] = 2
m["c"] = 3 // 可以继续写
```

给容量提示的意义是减少增长过程中的扩容和 rehash。它不会限制 map 只能放 `n` 个元素。

如果你知道大概要放多少元素，可以给提示：

```go
m := make(map[int]User, len(users))
for _, u := range users {
	m[u.ID] = u
}
```

如果只是几个元素，直接字面量更清楚：

```go
m := map[string]int{
	"go":  1,
	"sql": 2,
}
```

## 5. channel 缓冲大小应该怎么考虑？

无缓冲 channel 用来同步交接：

```go
ch := make(chan int)
```

发送方会等接收方准备好。它能提供天然背压。

有缓冲 channel 用来吸收短时间的生产消费速率差：

```go
ch := make(chan int, 100)
```

但缓冲不是越大越好。缓冲过大时，消费者变慢不会立刻暴露，队列里的任务会堆积，延迟和内存都会上升。

```go
func submit(ch chan<- Job, job Job) error {
	select {
	case ch <- job:
		return nil
	default:
		return fmt.Errorf("queue full")
	}
}
```

如果业务不能无限排队，就应该让缓冲满这个状态显式暴露出来，而不是盲目加大 channel。

## 6. `new` 和 `make` 是否决定对象在栈上还是堆上？

不决定。栈还是堆由逃逸分析决定。

```go
func heap() *int {
	x := 1
	return &x
}

func maybeStack() int {
	p := new(int)
	*p = 1
	return *p
}
```

第一个函数返回局部变量地址，`x` 必须在函数返回后仍然有效，所以会逃逸。第二个函数虽然用了 `new`，但如果编译器能证明指针不逃逸，就可能不产生真实堆分配。

验证命令：

```sh
go build -gcflags=-m ./...
```

回答这题时不要把 `new` 说成“堆分配”，那是常见误区。
