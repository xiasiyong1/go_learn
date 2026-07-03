# 002. map 并发安全

## 问题

Go 的 map 是否并发安全？如果多个 goroutine 读写 map 应该怎么处理？

## 先给结论

Go 的普通 map 不是并发安全容器。多个 goroutine 同时读写 map 时，问题不只是可能 panic，更本质的是哈希表内部状态会被并发修改，导致数据竞争和结构破坏。

## 深入理解

### 1. 这道题真正考察什么

- 是否知道“并发读通常没事”不等于“读写并发也没事”。
- 是否能根据访问模式选择 `Mutex`、`RWMutex`、`sync.Map` 或 channel 所有权模型。
- 是否能解释 map 写入可能触发扩容、迁移和 bucket 状态变化。
- 是否知道 race detector 可以发现数据竞争，但不能替你证明业务并发逻辑正确。

### 2. 底层机制要讲清楚

- map 底层有桶、溢出桶、哈希种子、装载因子和渐进扩容等状态，写操作可能改变这些状态。
- 并发写或读写并发会破坏运行时对 map 内部结构的一致性假设，所以运行时会尽量检测并抛出 `fatal error: concurrent map writes`。
- `sync.RWMutex` 保护的是临界区，不是 map 本身；锁的粒度和持锁时间决定吞吐。
- `sync.Map` 为特定模式优化，尤其是 key 稳定、读多写少、缓存类场景，不是普通 map 的无脑替代品。

### 3. 工程实践怎么取舍

- 读写逻辑复杂、需要维护多个字段一致性时，用普通 map 加锁更直观。
- 缓存 key 相对稳定、读远多于写，可以评估 `sync.Map`。
- 如果能把 map 所有权交给一个 goroutine，通过 channel 串行操作，逻辑会更容易推理。
- 高热点 key 场景可以考虑分片锁，但要接受复杂度和遍历成本。

### 4. 常见误区

- 只给写操作加锁，读操作不加锁，仍然是读写并发。
- 在锁外返回 map 内部的 slice、指针或可变对象，导致保护边界失效。
- 误以为 `RWMutex` 一定比 `Mutex` 快，忽视写锁等待、读锁数量和临界区长度。
- 为了消除 panic 改用 `recover`，这只是掩盖数据竞争。

## 如何验证理解

- 用 `go test -race` 验证是否存在数据竞争。
- 用 benchmark 对比 `Mutex`、`RWMutex`、`sync.Map` 和分片锁在目标读写比例下的表现。
- 用锁 profiling 观察是否出现明显锁竞争，再决定是否拆锁或换模型。

## 代码示例

```go
package main

import "sync"

type SafeMap struct {
	mu sync.RWMutex
	m  map[string]int
}

func NewSafeMap() *SafeMap {
	return &SafeMap{m: make(map[string]int)}
}

func (s *SafeMap) Get(key string) (int, bool) {
	s.mu.RLock()
	defer s.mu.RUnlock()

	v, ok := s.m[key]
	return v, ok
}

func (s *SafeMap) Set(key string, value int) {
	s.mu.Lock()
	defer s.mu.Unlock()

	s.m[key] = value
}
```

## 面试追问

追问参考答案：[follow-ups.md](follow-ups.md)

- 如果继续追问“map 并发安全”的底层机制，应该讲到哪些层次？
- 这个知识点在真实项目里应该如何取舍？
- 最容易踩的坑是什么？为什么这些坑不是背结论就能避免的？
- 如何用测试、工具或 profiling 验证自己的判断？
- 当数据量、并发量或团队规模变大后，这个问题会怎样升级？
