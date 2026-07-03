# 002. map 并发安全

## 问题

Go 的 map 是否并发安全？如果多个 goroutine 读写 map 应该怎么处理？

## 先给结论

Go 的普通 `map` 不是并发安全容器。

- 多个 goroutine 只读同一个 `map`，并且确定没有任何写入，通常是安全的。
- 只要存在并发写入，不管是写写并发，还是读写并发，都必须同步。
- 运行时有时会报 `fatal error: concurrent map writes` 或 `fatal error: concurrent map read and map write`，但没有报错也不代表代码正确，因为本质问题是数据竞争。

面试中不要只回答“map 不安全，用锁”。更好的回答是：先说明哪些访问组合不安全，再根据业务访问模式选择 `sync.Mutex`、`sync.RWMutex`、`sync.Map`、channel 所有权模型或分片锁。

## 为什么普通 map 不能并发读写

`map` 的写入不是简单地改一个值。写入过程中可能发生：

- 找 bucket、比较 key、写入 key/value。
- 维护哈希表的装载因子。
- 创建 overflow bucket。
- 触发扩容。
- 渐进式迁移旧 bucket 到新 bucket。

这些内部状态要求单线程视角下保持一致。并发读写时，一个 goroutine 可能正在看旧 bucket，另一个 goroutine 正在迁移或写入 bucket，读到的结构就可能处于中间状态。

下面这段代码是典型错误：

```go
package main

func main() {
	m := make(map[int]int)

	go func() {
		for i := 0; i < 100000; i++ {
			m[i] = i
		}
	}()

	for i := 0; i < 100000; i++ {
		_ = m[i]
	}
}
```

它可能直接崩溃，也可能在某些运行中“看起来没事”。面试时要强调：并发 bug 不能用“我跑过没报错”证明正确，要用同步原语建立 happens-before 关系，并用 `go test -race` 辅助检查。

## 方案一：普通 map 加锁

最常用、最容易解释、也最适合初学者掌握的方案，是把 `map` 和锁封装在同一个类型里。

```go
package cache

import "sync"

type CounterStore struct {
	mu sync.RWMutex
	m  map[string]int
}

func NewCounterStore() *CounterStore {
	return &CounterStore{
		m: make(map[string]int),
	}
}

func (s *CounterStore) Get(key string) (int, bool) {
	s.mu.RLock()
	defer s.mu.RUnlock()

	v, ok := s.m[key]
	return v, ok
}

func (s *CounterStore) Add(key string, delta int) int {
	s.mu.Lock()
	defer s.mu.Unlock()

	s.m[key] += delta
	return s.m[key]
}

func (s *CounterStore) Delete(key string) {
	s.mu.Lock()
	defer s.mu.Unlock()

	delete(s.m, key)
}
```

这里有两个关键点：

- 读也要加锁。只给写加锁，读不加锁，仍然是读写并发。
- 不要把内部 `map` 直接暴露出去。一旦返回了原始 `map`，调用方就能绕过锁修改它。

错误示例：

```go
func (s *CounterStore) Raw() map[string]int {
	s.mu.RLock()
	defer s.mu.RUnlock()

	return s.m // 错误：锁释放后，调用方仍然持有内部 map
}
```

正确做法通常是返回快照：

```go
func (s *CounterStore) Snapshot() map[string]int {
	s.mu.RLock()
	defer s.mu.RUnlock()

	cp := make(map[string]int, len(s.m))
	for k, v := range s.m {
		cp[k] = v
	}
	return cp
}
```

## 方案二：sync.Map

`sync.Map` 是标准库提供的并发安全 map，但它不是普通 `map` 的万能替代品。

它适合这类场景：

- 读多写少。
- key 相对稳定。
- 多 goroutine 访问互相独立的 key。
- 缓存、注册表、只追加或少删除的数据。

示例：

```go
package cache

import "sync"

type User struct {
	ID   string
	Name string
}

type UserCache struct {
	m sync.Map // key: string, value: *User
}

func (c *UserCache) Get(id string) (*User, bool) {
	v, ok := c.m.Load(id)
	if !ok {
		return nil, false
	}
	return v.(*User), true
}

func (c *UserCache) Set(u *User) {
	c.m.Store(u.ID, u)
}

func (c *UserCache) GetOrCreate(id string, load func(string) *User) *User {
	if v, ok := c.m.Load(id); ok {
		return v.(*User)
	}

	u := load(id)
	actual, _ := c.m.LoadOrStore(id, u)
	return actual.(*User)
}
```

注意这里没有直接写 `LoadOrStore(id, load(id))`，因为函数实参会先求值，那样即使缓存命中也会执行 `load(id)`。

这里还有一个容易忽略的点：`sync.Map` 只保证 map 这个容器的并发访问安全，不自动保证 value 指向的对象也安全。

```go
// 容器并发安全，不代表 *User 内部字段并发写安全。
u, _ := cache.Get("u1")
u.Name = "new name" // 如果多个 goroutine 同时改 User，仍然需要额外同步
```

如果 value 是可变对象，要么让它不可变，要么在对象内部继续加锁。

## 方案三：channel 所有权模型

如果业务能接受所有 map 操作都经过一个 goroutine，可以让这个 goroutine 独占 `map`。这样不需要锁，因为没有多个 goroutine 直接访问同一个 `map`。

```go
package cache

type getReq struct {
	key  string
	resp chan getResp
}

type getResp struct {
	value int
	ok    bool
}

type setReq struct {
	key   string
	value int
	done  chan struct{}
}

type OwnerMap struct {
	gets chan getReq
	sets chan setReq
}

func NewOwnerMap() *OwnerMap {
	o := &OwnerMap{
		gets: make(chan getReq),
		sets: make(chan setReq),
	}
	go o.loop()
	return o
}

func (o *OwnerMap) loop() {
	m := make(map[string]int)
	for {
		select {
		case req := <-o.gets:
			v, ok := m[req.key]
			req.resp <- getResp{value: v, ok: ok}
		case req := <-o.sets:
			m[req.key] = req.value
			close(req.done)
		}
	}
}

func (o *OwnerMap) Get(key string) (int, bool) {
	resp := make(chan getResp)
	o.gets <- getReq{key: key, resp: resp}
	r := <-resp
	return r.value, r.ok
}

func (o *OwnerMap) Set(key string, value int) {
	done := make(chan struct{})
	o.sets <- setReq{key: key, value: value, done: done}
	<-done
}
```

这个模型的优势是状态集中，推理简单；代价是所有操作会排队，吞吐取决于 owner goroutine 的处理能力。它更适合状态机、事件循环、连接会话状态等场景。

## 方案四：分片锁

如果已经通过 benchmark 或 profile 证明单把锁竞争严重，可以把一个大 map 拆成多个分片，每个分片一把锁。

```go
package cache

import (
	"hash/fnv"
	"sync"
)

const shardCount = 32

type shard struct {
	mu sync.RWMutex
	m  map[string]int
}

type ShardedMap struct {
	shards [shardCount]shard
}

func NewShardedMap() *ShardedMap {
	s := &ShardedMap{}
	for i := range s.shards {
		s.shards[i].m = make(map[string]int)
	}
	return s
}

func (s *ShardedMap) shardFor(key string) *shard {
	h := fnv.New32a()
	_, _ = h.Write([]byte(key))
	return &s.shards[h.Sum32()%shardCount]
}

func (s *ShardedMap) Set(key string, value int) {
	sh := s.shardFor(key)
	sh.mu.Lock()
	defer sh.mu.Unlock()

	sh.m[key] = value
}

func (s *ShardedMap) Get(key string) (int, bool) {
	sh := s.shardFor(key)
	sh.mu.RLock()
	defer sh.mu.RUnlock()

	v, ok := sh.m[key]
	return v, ok
}
```

分片锁不是初始方案。它会增加遍历、统计、删除、扩容和测试复杂度。面试中可以说：先用简单锁保证正确，再用压测数据决定是否分片。

## RWMutex 一定比 Mutex 快吗

不一定。

`RWMutex` 允许多个读者同时进入，但它也有额外管理成本。如果临界区很短、写入很频繁，或者读写比例没有明显偏读，`Mutex` 可能更简单也更快。

面试里比较稳的说法是：

- 先保证正确性。
- 根据读写比例和临界区耗时选择锁。
- 用 benchmark 和 mutex profile 验证，而不是凭感觉换成 `RWMutex`。

## 面试追问

追问参考答案：[follow-ups.md](follow-ups.md)

- “只读 map 是安全的”这句话有什么前提？
- 只给写操作加锁，读操作不加锁，为什么仍然错？
- `sync.Map` 和 `map` 加 `RWMutex` 应该怎么选？
- `sync.Map` 里的 value 是指针时，还需要考虑什么并发问题？
- 遍历 map 时如何保证不会读到一半被修改？
- 分片锁解决了什么问题，又引入了什么成本？
