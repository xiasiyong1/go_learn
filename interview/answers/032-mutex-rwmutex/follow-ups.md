# 032. Mutex 和 RWMutex - 面试追问

## 1. Mutex 保护的到底是什么？

保护的是共享状态和不变量。

```go
type Account struct {
	mu      sync.Mutex
	balance int64
}

func (a *Account) Withdraw(n int64) bool {
	a.mu.Lock()
	defer a.mu.Unlock()

	if a.balance < n {
		return false
	}
	a.balance -= n
	return true
}
```

这里锁保护的不只是 `balance -= n`，还保护“检查余额”和“扣减余额”必须作为一个整体完成。

如果把检查和扣减拆开加锁，中间可能被其他 goroutine 修改，业务不变量就破坏了。

## 2. RWMutex 一定比 Mutex 快吗？

不一定。

`RWMutex` 允许多个读者并发，但它有额外成本。临界区很短、写入频繁、竞争不明显时，`Mutex` 可能更快也更清楚。

应该用 benchmark 验证：

```go
func BenchmarkGet(b *testing.B) {
	for i := 0; i < b.N; i++ {
		cache.Get("key")
	}
}
```

选择原则：

- 正确性第一。
- 读多且读临界区有明显耗时时考虑 `RWMutex`。
- 没有数据前不要为了“看起来更并发”换锁。

## 3. 为什么读操作也需要加锁？

因为读写并发也是数据竞争。

错误示例：

```go
type Cache struct {
	mu sync.Mutex
	m  map[string]string
}

func (c *Cache) Get(key string) string {
	return c.m[key] // 没加锁
}

func (c *Cache) Set(key, value string) {
	c.mu.Lock()
	defer c.mu.Unlock()
	c.m[key] = value
}
```

写操作可能修改 map 内部结构，读操作不加锁仍然不安全。

正确写法：

```go
func (c *Cache) Get(key string) string {
	c.mu.Lock()
	defer c.mu.Unlock()
	return c.m[key]
}
```

或者使用 `RWMutex` 的 `RLock`。

## 4. 为什么不能在锁外返回内部 map 或 slice？

因为调用方会绕过锁修改内部状态。

```go
func (c *Cache) Raw() map[string]string {
	c.mu.RLock()
	defer c.mu.RUnlock()
	return c.m
}
```

调用方：

```go
cache.Raw()["x"] = "y"
```

这次写入没有经过锁。

正确做法是返回快照：

```go
func (c *Cache) Snapshot() map[string]string {
	c.mu.RLock()
	defer c.mu.RUnlock()

	cp := make(map[string]string, len(c.m))
	for k, v := range c.m {
		cp[k] = v
	}
	return cp
}
```

如果 map 的 value 是指针或 slice，还要继续考虑是否需要深拷贝。

## 5. 为什么持锁调用回调容易出问题？

因为你无法控制回调做什么。它可能很慢，可能再次请求同一把锁。

```go
func (c *Cache) Range(fn func(string, string)) {
	c.mu.RLock()
	defer c.mu.RUnlock()

	for k, v := range c.m {
		fn(k, v)
	}
}
```

如果 `fn` 里调用 `c.Set`，就可能阻塞等待写锁。

更好的做法：

```go
func (c *Cache) Range(fn func(string, string)) {
	items := c.Snapshot()
	for k, v := range items {
		fn(k, v)
	}
}
```

锁内复制数据，锁外执行未知逻辑。

## 6. 读锁能不能升级成写锁？

不能安全地直接升级。

错误示例：

```go
c.mu.RLock()
if _, ok := c.m[key]; !ok {
	c.mu.Lock() // 可能死锁
}
```

应该释放读锁，再获取写锁，并重新检查状态：

```go
c.mu.RLock()
_, ok := c.m[key]
c.mu.RUnlock()

if !ok {
	c.mu.Lock()
	defer c.mu.Unlock()

	if _, ok := c.m[key]; !ok {
		c.m[key] = value
	}
}
```

重新检查是必要的，因为释放读锁到获得写锁之间，其他 goroutine 可能已经写入。
