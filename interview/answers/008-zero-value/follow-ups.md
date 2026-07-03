# 008. 零值 - 面试追问

## 1. nil slice、nil map、nil channel 的零值行为分别是什么？

三者都是 `nil`，但可用边界不同。

```go
var s []int
fmt.Println(len(s)) // 0
s = append(s, 1)    // 可以

var m map[string]int
fmt.Println(m["go"]) // 0，可以读
// m["go"] = 1       // panic，不能写

var ch chan int
// ch <- 1           // 永久阻塞
// <-ch              // 永久阻塞
```

面试时可以这样总结：

- nil slice 像空列表，可以追加。
- nil map 像未初始化的哈希表，可以查但不能写。
- nil channel 没有通信对象，收发都无法完成。

## 2. 为什么 nil slice 可以 `append`，nil map 写入会 panic？

slice 是一个描述符，包含数据指针、长度、容量。nil slice 的描述符是合法的，只是还没有底层数组。`append` 发现容量不够时，会分配新数组。

```go
var s []int
s = append(s, 10)
fmt.Println(s) // [10]
```

map 的写入需要真实的哈希表结构。nil map 没有这个结构，运行时不知道该写到哪里，所以会 panic。

```go
var m map[string]int
m["go"] = 10 // panic
```

正确写法：

```go
m := make(map[string]int)
m["go"] = 10
```

如果希望结构体零值可用，可以在方法里延迟初始化 map：

```go
func (c *Counter) Add(key string) {
	if c.m == nil {
		c.m = make(map[string]int)
	}
	c.m[key]++
}
```

## 3. nil channel 在 `select` 中有什么用，误用有什么风险？

nil channel 的收发永远不会就绪。在 `select` 中，这可以用来动态关闭某个分支。

```go
func readBoth(a, b <-chan int) {
	for a != nil || b != nil {
		select {
		case v, ok := <-a:
			if !ok {
				a = nil
				continue
			}
			fmt.Println(v)
		case v, ok := <-b:
			if !ok {
				b = nil
				continue
			}
			fmt.Println(v)
		}
	}
}
```

当 `a` 关闭后，把 `a = nil`，对应 case 就不会再被选中。

风险是：如果你不是刻意这样设计，nil channel 可能让 goroutine 永久阻塞。

```go
type Worker struct {
	jobs chan Job
}

func (w *Worker) Submit(job Job) {
	w.jobs <- job // 如果 jobs 没初始化，会永久阻塞
}
```

这种类型通常应该通过构造函数初始化 channel，或者在方法里明确检查。

## 4. 标准库里哪些类型体现了零值可用？

典型例子是 `sync.Mutex`、`bytes.Buffer`、`strings.Builder`。

```go
var mu sync.Mutex
mu.Lock()
mu.Unlock()

var buf bytes.Buffer
buf.WriteString("hello")
fmt.Println(buf.String())
```

这些类型不要求调用方先调用 `Init` 或 `New`，降低了使用成本。

但要注意：零值可用不代表可以随便复制。

```go
type Counter struct {
	mu sync.Mutex
	n  int
}
```

`var c Counter` 可以直接用；但 `c` 一旦被多个 goroutine 使用，就不应该复制它。复制后可能出现两个锁保护同一份逻辑状态，或者把锁的内部状态复制出去。

## 5. 什么时候应该追求零值可用，什么时候应该提供构造函数？

适合追求零值可用：

- 空状态有明确含义。
- 没有必填配置。
- 不依赖外部资源。
- 方法可以自然处理 nil 字段。

```go
type Set struct {
	items map[string]struct{}
}

func (s *Set) Add(v string) {
	if s.items == nil {
		s.items = make(map[string]struct{})
	}
	s.items[v] = struct{}{}
}
```

适合构造函数：

- 必须传配置。
- 必须打开连接或创建 channel。
- 必须校验参数。
- 零值无法表示合法状态。

```go
func NewWorker(size int) (*Worker, error) {
	if size <= 0 {
		return nil, fmt.Errorf("size must be positive")
	}
	return &Worker{
		jobs: make(chan Job, size),
	}, nil
}
```

面试里可以说：零值可用是好设计，但不是强行不用构造函数。

## 6. 为什么“零值可用”不等于“不需要考虑初始化”？

因为零值只保证变量处于确定状态，不保证所有业务操作都有意义。

```go
type Client struct {
	baseURL string
	http    *http.Client
}

func (c *Client) Get(path string) (*http.Response, error) {
	return c.http.Get(c.baseURL + path) // 零值会 panic
}
```

这里 `Client` 的零值里 `http == nil`，直接调用会 panic。更好的设计是构造函数：

```go
func NewClient(baseURL string) *Client {
	return &Client{
		baseURL: baseURL,
		http:    &http.Client{Timeout: 3 * time.Second},
	}
}
```

或者方法里提供默认值：

```go
func (c *Client) httpClient() *http.Client {
	if c.http != nil {
		return c.http
	}
	return http.DefaultClient
}
```

选择哪种方式取决于业务：如果 `baseURL` 必填，构造函数更清楚；如果缺省值有合理含义，零值可用更方便。
