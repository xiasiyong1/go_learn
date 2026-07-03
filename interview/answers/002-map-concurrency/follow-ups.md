# 002. map 并发安全 - 面试追问

## 1. “只读 map 是安全的”这句话有什么前提？

前提是：所有 goroutine 都只读，并且在这些读发生期间没有任何 goroutine 写这个 `map`。

下面这种写法是可以接受的：先构造好配置表，再只读使用。

```go
var statusText = map[int]string{
	200: "ok",
	404: "not found",
	500: "server error",
}

func Text(code int) string {
	return statusText[code]
}
```

但是下面这种写法不安全，因为后台 goroutine 可能更新 `routes`，前台 goroutine 同时读取：

```go
var routes = map[string]string{}

func Read(path string) string {
	return routes[path] // 没加锁的读
}

func Reload(newRoutes map[string]string) {
	for k, v := range newRoutes {
		routes[k] = v // 没加锁的写
	}
}
```

面试官问这句时，真正想看的是你能否补上“没有并发写”这个条件，而不是机械背“map 不能并发”。

## 2. 只给写操作加锁，读操作不加锁，为什么仍然错？

因为读和写访问的是同一个 `map` 内部结构。写入可能改变 bucket、overflow bucket、扩容状态；读操作如果没有锁，就可能在写操作进行到一半时观察到不一致的内部状态。

错误示例：

```go
type Store struct {
	mu sync.Mutex
	m  map[string]int
}

func (s *Store) Get(key string) int {
	return s.m[key] // 错误：读没有和写建立同步关系
}

func (s *Store) Set(key string, value int) {
	s.mu.Lock()
	defer s.mu.Unlock()

	s.m[key] = value
}
```

正确写法：

```go
type Store struct {
	mu sync.RWMutex
	m  map[string]int
}

func (s *Store) Get(key string) (int, bool) {
	s.mu.RLock()
	defer s.mu.RUnlock()

	v, ok := s.m[key]
	return v, ok
}

func (s *Store) Set(key string, value int) {
	s.mu.Lock()
	defer s.mu.Unlock()

	s.m[key] = value
}
```

这里的关键不是“读锁比写锁轻”，而是所有读写都必须经过同一个同步边界。

## 3. `sync.Map` 和 `map` 加 `RWMutex` 应该怎么选？

可以按访问模式选。

普通 `map` 加锁更适合：

- 需要维护多个字段或多个 map 的一致性。
- 需要强类型 API。
- 有复杂遍历、统计、批量更新逻辑。
- 写入不少，或者读写比例不确定。

`sync.Map` 更适合：

- 读多写少。
- key 集合相对稳定。
- 多 goroutine 访问不同 key。
- 缓存、注册表、单例对象表。

例如需要“扣库存同时写流水”，不能只把库存放进 `sync.Map` 就结束，因为业务一致性跨越多个状态：

```go
type Inventory struct {
	mu     sync.Mutex
	stock  map[string]int
	events []string
}

func (i *Inventory) Deduct(sku string, n int) bool {
	i.mu.Lock()
	defer i.mu.Unlock()

	if i.stock[sku] < n {
		return false
	}
	i.stock[sku] -= n
	i.events = append(i.events, sku)
	return true
}
```

这类场景用一把锁保护“库存和流水的一致性”更清楚。`sync.Map` 解决的是容器并发访问，不解决跨对象事务。

## 4. `sync.Map` 里的 value 是指针时，还需要考虑什么？

`sync.Map` 只保护容器的 `Load`、`Store`、`Delete` 等操作，不保护 value 指向对象内部的字段。

错误示例：

```go
type User struct {
	Name string
}

var users sync.Map // key: string, value: *User

func Rename(id, name string) {
	v, _ := users.Load(id)
	u := v.(*User)
	u.Name = name // 多 goroutine 同时改同一个 User 时仍然数据竞争
}
```

可以把 value 设计成不可变对象，每次更新都替换整个指针：

```go
type User struct {
	Name string
}

func Rename(id, name string) {
	users.Store(id, &User{Name: name})
}
```

如果对象字段很多、需要原地修改，就给对象内部加锁：

```go
type User struct {
	mu   sync.Mutex
	Name string
}

func Rename(id, name string) {
	v, _ := users.Load(id)
	u := v.(*User)

	u.mu.Lock()
	defer u.mu.Unlock()
	u.Name = name
}
```

面试里这道追问很常见，因为很多人只记住了“`sync.Map` 并发安全”，忽略了它保护的边界。

## 5. 遍历 map 时如何避免一边遍历一边修改？

如果使用普通 `map` 加锁，遍历时最简单的做法是在锁内复制快照，然后在锁外做耗时处理。

```go
func (s *Store) Items() map[string]int {
	s.mu.RLock()
	defer s.mu.RUnlock()

	items := make(map[string]int, len(s.m))
	for k, v := range s.m {
		items[k] = v
	}
	return items
}
```

不要在持锁期间调用外部回调，除非你能控制回调不会阻塞、不会反过来调用 `Store`、不会造成死锁。

容易出问题的写法：

```go
func (s *Store) Range(fn func(string, int)) {
	s.mu.RLock()
	defer s.mu.RUnlock()

	for k, v := range s.m {
		fn(k, v) // 回调可能很慢，也可能反过来调用 Set 导致死锁
	}
}
```

如果必须提供 `Range` 风格 API，可以先复制 key/value，再执行回调：

```go
func (s *Store) Range(fn func(string, int)) {
	items := s.Items()
	for k, v := range items {
		fn(k, v)
	}
}
```

这个写法牺牲了一些内存，但锁持有时间更可控。

## 6. 分片锁解决了什么问题，又引入了什么成本？

分片锁解决的是单把锁竞争：不同 key 分到不同 shard，就可以并发读写不同 shard。

但是它会引入这些成本：

- 每次访问都要计算 shard。
- 全量遍历需要遍历所有 shard。
- 需要跨 shard 维护全局统计时更复杂。
- 热点 key 如果集中在一个 shard，竞争仍然存在。

例如统计总数时，必须依次拿每个 shard 的读锁：

```go
func (s *ShardedMap) Len() int {
	total := 0
	for i := range s.shards {
		sh := &s.shards[i]
		sh.mu.RLock()
		total += len(sh.m)
		sh.mu.RUnlock()
	}
	return total
}
```

如果面试官继续问“是不是分片越多越好”，答案是否定的。分片越多，单 shard 竞争可能下降，但内存、初始化、遍历和统计成本会上升。实际项目里要用 benchmark 和 profile 做决定。
