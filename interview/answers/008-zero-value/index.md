# 008. 零值

## 问题

Go 常见类型的零值是什么？零值可用有什么意义？

## 先给结论

Go 里变量声明后即使没有显式初始化，也会得到确定的零值，不会是随机内存。

常见零值：

| 类型 | 零值 |
| --- | --- |
| `int`、`int64`、`float64` 等数值类型 | `0` |
| `bool` | `false` |
| `string` | `""` |
| 指针 | `nil` |
| slice | `nil` |
| map | `nil` |
| channel | `nil` |
| function | `nil` |
| interface | `nil` |
| struct | 每个字段都是对应类型零值 |

“零值可用”是 Go 很重要的设计风格：很多类型不需要构造函数，声明出来就能用。但这不等于所有零值都支持所有操作。nil slice 可以 `append`，nil map 不能写，nil channel 收发会永久阻塞。

## 基础类型零值

```go
var n int
var ok bool
var s string
var p *int

fmt.Println(n)      // 0
fmt.Println(ok)     // false
fmt.Println(s == "") // true
fmt.Println(p == nil) // true
```

struct 的零值是字段零值组合：

```go
type User struct {
	ID   int64
	Name string
	Tags []string
}

var u User
fmt.Println(u.ID)        // 0
fmt.Println(u.Name == "") // true
fmt.Println(u.Tags == nil) // true
```

## nil slice：可以读、遍历、append

```go
var xs []int

fmt.Println(xs == nil) // true
fmt.Println(len(xs))   // 0

for _, x := range xs {
	fmt.Println(x) // 不会执行，也不会 panic
}

xs = append(xs, 1)
fmt.Println(xs) // [1]
```

nil slice 没有底层数组，但 `append` 会按需分配，所以它常常是很自然的零值。

这就是为什么很多函数可以这样写：

```go
func Even(nums []int) []int {
	var out []int
	for _, n := range nums {
		if n%2 == 0 {
			out = append(out, n)
		}
	}
	return out
}
```

没有偶数时返回 nil slice，调用方仍然可以 `len`、`range`、`append`。

## nil map：可以读，不能写

```go
var scores map[string]int

fmt.Println(scores == nil) // true
fmt.Println(scores["go"])  // 0

scores["go"] = 100 // panic: assignment to entry in nil map
```

nil map 还没有初始化哈希表结构，所以不能写入。写之前必须 `make`：

```go
scores := make(map[string]int)
scores["go"] = 100
```

如果 struct 里有 map 字段，零值方法里要注意初始化：

```go
type Counter struct {
	values map[string]int
}

func (c *Counter) Add(key string) {
	if c.values == nil {
		c.values = make(map[string]int)
	}
	c.values[key]++
}
```

这样 `var c Counter; c.Add("go")` 就能正常工作，类型更符合零值可用的习惯。

## nil channel：收发都会阻塞

```go
var ch chan int

// ch <- 1  // 永久阻塞
// <-ch     // 永久阻塞
```

nil channel 最常见的用途是在 `select` 中动态禁用某个 case。

```go
func merge(a, b <-chan int) <-chan int {
	out := make(chan int)
	go func() {
		defer close(out)

		for a != nil || b != nil {
			select {
			case v, ok := <-a:
				if !ok {
					a = nil // 禁用这个 case
					continue
				}
				out <- v
			case v, ok := <-b:
				if !ok {
					b = nil
					continue
				}
				out <- v
			}
		}
	}()
	return out
}
```

如果你不是刻意用 nil channel 禁用 case，就要小心它可能导致 goroutine 永久阻塞。

## 零值可用的标准库类型

很多标准库类型专门设计成零值可用。

```go
var mu sync.Mutex
mu.Lock()
mu.Unlock()

var buf bytes.Buffer
buf.WriteString("hello")
fmt.Println(buf.String())
```

`sync.Mutex` 不需要 `NewMutex`；`bytes.Buffer` 不需要先调用 `Init`。这让类型使用更简单，也减少了不必要的构造函数。

但有些类型不能复制使用，尤其是使用后不能复制：

```go
type SafeCounter struct {
	mu sync.Mutex
	n  int
}
```

`SafeCounter` 的零值可以用，但一旦用过，通常不要复制这个结构体，否则锁状态和保护的数据可能被复制出问题。

## 设计自己的类型时如何利用零值

推荐让零值处于合法状态：

```go
type Logger struct {
	out io.Writer
}

func (l *Logger) Print(msg string) {
	if l.out == nil {
		l.out = io.Discard
	}
	fmt.Fprintln(l.out, msg)
}
```

这样调用方可以直接：

```go
var l Logger
l.Print("hello")
```

但不是所有类型都应该硬凑零值可用。如果类型必须依赖外部资源或必填配置，构造函数更清楚：

```go
type Client struct {
	baseURL string
	http    *http.Client
}

func NewClient(baseURL string) *Client {
	return &Client{
		baseURL: baseURL,
		http:    &http.Client{Timeout: 3 * time.Second},
	}
}
```

这里 `baseURL` 是必填约束，用构造函数表达比让零值半可用更好。

## 常见判断方式

判断集合是否为空，通常用 `len`：

```go
if len(items) == 0 {
	return nil
}
```

不要用 `items == nil` 判断“有没有数据”，因为 nil slice 和 empty slice 都表示长度为 0：

```go
var a []int
b := []int{}

fmt.Println(len(a) == 0) // true
fmt.Println(len(b) == 0) // true
fmt.Println(a == nil)   // true
fmt.Println(b == nil)   // false
```

只有当业务上确实要区分“未初始化”和“已初始化但为空”时，才应该使用 `nil` 判断。

## 面试追问

追问参考答案：[follow-ups.md](follow-ups.md)

- nil slice、nil map、nil channel 的零值行为分别是什么？
- 为什么 nil slice 可以 `append`，nil map 写入会 panic？
- nil channel 在 `select` 中有什么用，误用有什么风险？
- 标准库里哪些类型体现了零值可用？
- 设计结构体时，什么时候应该追求零值可用，什么时候应该提供构造函数？
- 为什么“零值可用”不等于“不需要考虑初始化”？
