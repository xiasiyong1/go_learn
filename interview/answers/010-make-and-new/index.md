# 010. make 和 new

## 问题

`make` 和 `new` 有什么区别？

## 先给结论

`new(T)` 分配一个 `T` 类型的零值，并返回 `*T`。

`make` 只用于 slice、map、channel，负责初始化它们背后的运行时结构，并返回类型本身，不返回指针。

```go
p := new(int)             // *int
s := make([]int, 0, 10)   // []int
m := make(map[string]int) // map[string]int
ch := make(chan int, 1)   // chan int
```

核心区别不是“堆和栈”，而是：

- `new`：给任意类型分配零值。
- `make`：创建 slice、map、channel 可用的运行时结构。

对象到底在栈上还是堆上，由逃逸分析决定，不由 `new` 或 `make` 这个关键字直接决定。

## new 做了什么

`new(T)` 得到的是 `*T`，指向一个零值 `T`。

```go
p := new(int)
fmt.Println(*p) // 0

*p = 10
fmt.Println(*p) // 10
```

对结构体也一样：

```go
type User struct {
	Name string
	Age  int
}

u := new(User)
fmt.Println(u.Name == "") // true
fmt.Println(u.Age)        // 0
```

实际项目里更常见的是复合字面量，因为字段含义更清楚：

```go
u := &User{
	Name: "Tom",
	Age:  18,
}
```

`new(User)` 和 `&User{}` 都能得到 `*User`，但 `&User{Field: value}` 可读性通常更好。

## make 做了什么

`make` 只能用于三类类型：

```go
make([]int, 0, 10)
make(map[string]int, 100)
make(chan Job, 10)
```

原因是这三类类型的零值虽然确定，但还没有完整的运行时结构：

- slice 需要描述底层数组、长度、容量。
- map 需要哈希表结构。
- channel 需要队列、缓冲区和等待队列。

## new(map) 的坑

这是面试里最常见的反例。

```go
p := new(map[string]int)
fmt.Println(*p == nil) // true

(*p)["go"] = 1 // panic: assignment to entry in nil map
```

`new(map[string]int)` 只是创建了一个零值 map，而零值 map 是 nil。它并没有初始化哈希表。

正确写法：

```go
m := make(map[string]int)
m["go"] = 1
```

如果你确实需要 `*map[string]int`，也要先 `make`：

```go
m := make(map[string]int)
p := &m
(*p)["go"] = 1
```

不过多数场景不需要 `*map`，map 本身就是引用语义的描述符。

## make slice 的 len 和 cap

`make([]T, len)` 创建的是长度为 `len` 的切片，里面已经有 `len` 个零值元素。

```go
s := make([]int, 3)
fmt.Println(s) // [0 0 0]

s = append(s, 1)
fmt.Println(s) // [0 0 0 1]
```

如果你的目标是“预计追加 3 个元素”，应该写成：

```go
s := make([]int, 0, 3)
s = append(s, 1)
fmt.Println(s) // [1]
```

选择规则：

- 已经有 n 个元素位置要填写：`make([]T, n)`。
- 现在为空，只是预计追加 n 个：`make([]T, 0, n)`。

例如按下标写入：

```go
ids := make([]int64, len(users))
for i, u := range users {
	ids[i] = u.ID
}
```

按 append 收集：

```go
ids := make([]int64, 0, len(users))
for _, u := range users {
	ids = append(ids, u.ID)
}
```

## make map 的容量提示

```go
m := make(map[string]int, len(items))
for _, item := range items {
	m[item.Name] = item.Count
}
```

第二个参数是容量提示，不是固定容量。map 仍然可以继续增长。给合理提示可以减少扩容和 rehash，但不需要为了小 map 过度计算容量。

## make channel 的缓冲大小

```go
jobs := make(chan Job)     // 无缓冲 channel
queue := make(chan Job, 8) // 有缓冲 channel
```

无缓冲 channel 强调同步交接：发送和接收必须同时准备好。

有缓冲 channel 允许短暂排队，但缓冲不是越大越好。过大的缓冲可能掩盖消费者变慢，让延迟和内存上涨更晚暴露。

```go
func StartWorker(buffer int) chan<- Job {
	jobs := make(chan Job, buffer)
	go func() {
		for job := range jobs {
			handle(job)
		}
	}()
	return jobs
}
```

这里的 `buffer` 应该来自业务吞吐和背压设计，而不是随便写一个很大的数字。

## new 和 make 不决定栈堆

错误理解：`new` 一定在堆上，普通变量一定在栈上。

实际由逃逸分析决定：

```go
func A() *int {
	x := 1
	return &x // x 逃逸，通常会放到堆上
}

func B() int {
	p := new(int)
	*p = 1
	return *p // 如果不逃逸，编译器可能优化掉堆分配
}
```

可以用命令观察：

```sh
go build -gcflags=-m ./...
```

面试里说清楚这点，能避免把 `new/make` 背成 C 语言里的 malloc。

## 面试追问

追问参考答案：[follow-ups.md](follow-ups.md)

- `new(T)` 的返回值是什么？它是否会初始化 map/channel？
- 为什么 `new(map[string]int)` 后写入仍然 panic？
- `make([]int, 3)` 和 `make([]int, 0, 3)` 的区别是什么？
- `make(map[K]V, n)` 的 `n` 是长度还是容量？
- channel 缓冲大小应该怎么考虑？
- `new` 和 `make` 是否决定对象在栈上还是堆上？
