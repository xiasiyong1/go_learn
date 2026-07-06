# 032. Mutex 和 RWMutex

## 问题

`Mutex` 和 `RWMutex` 有什么区别？如何避免锁误用？

## 先给结论

`sync.Mutex` 提供互斥访问，同一时刻只有一个 goroutine 能进入临界区。

`sync.RWMutex` 允许多个读者同时进入，但写者需要独占。它适合读多写少且读临界区有一定耗时的场景；它不一定比 `Mutex` 快。

锁保护的不是某一行代码，而是共享状态和不变量。真正要设计清楚的是：

- 哪些数据必须被锁保护。
- 哪些操作必须在同一个临界区里完成。
- 锁内不能做哪些慢操作或未知调用。

## Mutex 基本写法

```go
type Counter struct {
	mu sync.Mutex
	n  int
}

func (c *Counter) Inc() {
	c.mu.Lock()
	defer c.mu.Unlock()

	c.n++
}

func (c *Counter) Value() int {
	c.mu.Lock()
	defer c.mu.Unlock()

	return c.n
}
```

读也要加锁。只给写加锁，读不加锁，仍然是数据竞争。

## RWMutex 基本写法

```go
type Cache struct {
	mu sync.RWMutex
	m  map[string]string
}

func (c *Cache) Get(key string) (string, bool) {
	c.mu.RLock()
	defer c.mu.RUnlock()

	v, ok := c.m[key]
	return v, ok
}

func (c *Cache) Set(key, value string) {
	c.mu.Lock()
	defer c.mu.Unlock()

	c.m[key] = value
}
```

`RLock` 可以被多个读者同时持有。`Lock` 需要排斥所有读者和写者。

## RWMutex 不一定更快

`RWMutex` 有额外管理成本。如果临界区很短、写入频繁，`Mutex` 可能更简单也更快。

不应该只凭“读多写少”就改：

```go
// 先 benchmark，再决定。
go test -bench . -benchmem
```

更稳的选择路径：

- 先用简单 `Mutex` 保证正确。
- 如果 profile 显示锁竞争明显，再分析读写比例和临界区耗时。
- 读多且读临界区较长，再考虑 `RWMutex`。

## 不要在锁外返回内部可变引用

错误示例：

```go
func (c *Cache) Items() map[string]string {
	c.mu.RLock()
	defer c.mu.RUnlock()

	return c.m
}
```

调用方拿到内部 map 后，可以绕过锁修改：

```go
items := cache.Items()
items["x"] = "y" // 锁保护失效
```

正确做法是返回快照：

```go
func (c *Cache) Items() map[string]string {
	c.mu.RLock()
	defer c.mu.RUnlock()

	cp := make(map[string]string, len(c.m))
	for k, v := range c.m {
		cp[k] = v
	}
	return cp
}
```

如果 value 是 slice、map、指针，也要考虑深拷贝或不可变设计。

## 不要持锁调用未知函数

错误示例：

```go
func (c *Cache) Range(fn func(string, string)) {
	c.mu.RLock()
	defer c.mu.RUnlock()

	for k, v := range c.m {
		fn(k, v)
	}
}
```

`fn` 可能很慢，可能反过来调用 `Set`，导致死锁或长时间持锁。

更安全的做法是先复制快照：

```go
func (c *Cache) Range(fn func(string, string)) {
	items := c.Items()
	for k, v := range items {
		fn(k, v)
	}
}
```

锁内只做必要的共享状态访问，锁外做慢操作。

## 不能从读锁升级到写锁

错误示例：

```go
func (c *Cache) GetOrSet(key, value string) string {
	c.mu.RLock()
	if v, ok := c.m[key]; ok {
		c.mu.RUnlock()
		return v
	}

	c.mu.Lock() // 错误：仍持有读锁时尝试写锁，容易死锁
	defer c.mu.Unlock()
	defer c.mu.RUnlock()

	c.m[key] = value
	return value
}
```

正确写法是释放读锁后再拿写锁，并在写锁下重新检查：

```go
func (c *Cache) GetOrSet(key, value string) string {
	c.mu.RLock()
	v, ok := c.m[key]
	c.mu.RUnlock()
	if ok {
		return v
	}

	c.mu.Lock()
	defer c.mu.Unlock()

	if v, ok := c.m[key]; ok {
		return v
	}
	c.m[key] = value
	return value
}
```

释放读锁到获取写锁之间，其他 goroutine 可能已经写入，所以必须 double check。

## 含锁结构体不要复制

```go
type Store struct {
	mu sync.Mutex
	m  map[string]string
}

func Bad(s Store) {
	s.mu.Lock()
	defer s.mu.Unlock()
}
```

复制已经使用的锁是错误设计。用指针接收者和指针传参：

```go
func Good(s *Store) {
	s.mu.Lock()
	defer s.mu.Unlock()
}
```

工具：

```sh
go vet -copylocks ./...
```

## 面试追问

追问参考答案：[follow-ups.md](follow-ups.md)

- Mutex 保护的到底是什么？
- RWMutex 一定比 Mutex 快吗？
- 为什么读操作也需要加锁？
- 为什么不能在锁外返回内部 map 或 slice？
- 为什么持锁调用回调容易出问题？
- 读锁能不能升级成写锁？
