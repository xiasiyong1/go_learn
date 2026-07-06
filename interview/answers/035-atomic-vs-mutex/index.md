# 035. atomic 和 Mutex

## 问题

原子操作和互斥锁如何选择？

## 先给结论

`sync/atomic` 适合对单个变量做简单、独立的并发读写，比如计数器、开关、状态位、只读配置指针替换。

`sync.Mutex` 适合保护一段临界区，尤其是多个字段必须一起读写、必须维护业务不变量时。

经验是：能用锁清楚表达正确性时，先用锁。只有在热点路径里确认锁竞争明显，并且状态足够简单时，再考虑 atomic。

## atomic 计数器

```go
type Metrics struct {
	requests atomic.Int64
}

func (m *Metrics) Inc() {
	m.requests.Add(1)
}

func (m *Metrics) Requests() int64 {
	return m.requests.Load()
}
```

所有访问都要通过 atomic。如果一边 atomic 写，一边普通读，仍然是数据竞争。

错误示例：

```go
var n int64

func Inc() {
	atomic.AddInt64(&n, 1)
}

func Read() int64 {
	return n // 错误：普通读
}
```

推荐使用 `atomic.Int64` 这类类型，避免散落的函数调用。

## atomic 开关

```go
type Server struct {
	closing atomic.Bool
}

func (s *Server) Close() {
	s.closing.Store(true)
}

func (s *Server) Handle() error {
	if s.closing.Load() {
		return ErrClosing
	}
	return nil
}
```

这类单个布尔状态很适合 atomic。

## 多字段不变量用 Mutex

错误方向：

```go
type Account struct {
	balance atomic.Int64
	frozen  atomic.Bool
}

func (a *Account) Withdraw(n int64) bool {
	if a.frozen.Load() {
		return false
	}
	if a.balance.Load() < n {
		return false
	}
	a.balance.Add(-n)
	return true
}
```

这里 `frozen` 和 `balance` 之间没有事务关系。检查后到扣款前，状态可能被其他 goroutine 改掉。

用锁更清楚：

```go
type Account struct {
	mu      sync.Mutex
	balance int64
	frozen  bool
}

func (a *Account) Withdraw(n int64) bool {
	a.mu.Lock()
	defer a.mu.Unlock()

	if a.frozen || a.balance < n {
		return false
	}
	a.balance -= n
	return true
}
```

锁保护的是“冻结状态、余额检查、扣款”这个整体不变量。

## 配置快照可以用 atomic.Value

适合读多写少、整体替换不可变快照：

```go
type Config struct {
	Timeout time.Duration
	Limit   int
}

var current atomic.Value // stores Config

func LoadConfig() Config {
	return current.Load().(Config)
}

func UpdateConfig(cfg Config) {
	current.Store(cfg)
}
```

注意：存进去的配置最好视为不可变。如果里面有 map 或 slice，要复制后再 Store，避免读者看到被修改中的数据。

## atomic 不等于一定快

atomic 可以减少锁阻塞，但它也可能带来：

- 忙等浪费 CPU。
- 复杂内存可见性推理。
- 多变量不一致。
- 代码可读性下降。

不推荐为了“无锁”写复杂循环：

```go
for !ready.Load() {
	// busy wait
}
```

更好的方式通常是 channel、cond、context 或 mutex，取决于场景。

## 面试追问

追问参考答案：[follow-ups.md](follow-ups.md)

- atomic 适合哪些简单场景？
- 为什么 atomic 写、普通读仍然有数据竞争？
- 多个 atomic 变量能不能维护复杂不变量？
- `atomic.Value` 适合什么场景？
- 为什么 Mutex 往往更容易写对？
- atomic 一定比 Mutex 快吗？
