# 035. atomic 和 Mutex - 面试追问

## 1. atomic 适合哪些简单场景？

适合单变量、独立状态。

计数器：

```go
var requests atomic.Int64

requests.Add(1)
fmt.Println(requests.Load())
```

开关：

```go
var closed atomic.Bool

closed.Store(true)
if closed.Load() {
	return ErrClosed
}
```

配置快照指针：

```go
var cfg atomic.Value // stores *Config
cfg.Store(&Config{Timeout: time.Second})
```

如果状态之间有业务关系，就不要急着用 atomic。

## 2. 为什么 atomic 写、普通读仍然有数据竞争？

因为同步必须覆盖所有访问路径。

错误示例：

```go
var n int64

func Inc() {
	atomic.AddInt64(&n, 1)
}

func Get() int64 {
	return n
}
```

`Get` 是普通读，没有使用 atomic。正确写法：

```go
func Get() int64 {
	return atomic.LoadInt64(&n)
}
```

使用 `atomic.Int64` 更不容易写错：

```go
var n atomic.Int64

func Inc() { n.Add(1) }
func Get() int64 { return n.Load() }
```

## 3. 多个 atomic 变量能不能维护复杂不变量？

通常不适合。

```go
type State struct {
	ready atomic.Bool
	count atomic.Int64
}
```

你可能读到 `ready=true` 但 `count` 还没更新，或者反过来。多个 atomic 变量之间没有自动事务。

需要一致快照时，用锁：

```go
type State struct {
	mu    sync.Mutex
	ready bool
	count int64
}

func (s *State) Snapshot() (bool, int64) {
	s.mu.Lock()
	defer s.mu.Unlock()
	return s.ready, s.count
}
```

或者把不可变快照整体替换：

```go
type Snapshot struct {
	Ready bool
	Count int64
}

var snap atomic.Value // stores Snapshot
```

## 4. `atomic.Value` 适合什么场景？

适合读多写少的整体替换。

```go
type Config struct {
	Limit int
}

var current atomic.Value // stores Config

func GetConfig() Config {
	return current.Load().(Config)
}

func SetConfig(c Config) {
	current.Store(c)
}
```

注意不要 Store 一个会被继续修改的 map：

```go
type Config struct {
	Headers map[string]string
}
```

如果要存这种配置，先深拷贝，让读者看到不可变快照。

## 5. 为什么 Mutex 往往更容易写对？

因为它能把多个操作放进同一个临界区。

```go
func (a *Account) Transfer(to *Account, n int64) bool {
	a.mu.Lock()
	defer a.mu.Unlock()

	if a.balance < n {
		return false
	}
	a.balance -= n
	to.balance += n
	return true
}
```

这里的检查和更新必须一起完成。用 atomic 拆成多个变量，很难证明没有中间态。

锁的代码通常更容易 review：锁住、检查、修改、解锁。

## 6. atomic 一定比 Mutex 快吗？

不一定。

低竞争场景下，Mutex 成本可能很低；atomic 如果写成忙等或复杂 CAS 循环，可能更慢也更难维护。

判断方式是 benchmark 和 profile：

```sh
go test -bench . -benchmem
go test -run '^$' -bench . -mutexprofile mutex.out
```

面试回答要稳：先选能证明正确的方案，再用数据决定是否优化。
