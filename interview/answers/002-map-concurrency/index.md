# 002. map 并发安全

## 问题

Go 的 map 是否并发安全？如果多个 goroutine 读写 map 应该怎么处理？

## 答案

Go 内置 `map` 不是并发安全的。多个 goroutine 同时读写同一个 `map`，可能触发 `fatal error: concurrent map read and map write`，也可能产生数据竞争。

常见处理方式：

- 用 `sync.RWMutex` 保护普通 `map`，适合需要维护复杂读写逻辑的场景。
- 使用 `sync.Map`，适合读多写少、key 稳定、缓存类场景。
- 通过 channel 串行化所有 map 操作，适合所有权模型清晰的场景。

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

- `sync.Map` 和 `map` 加锁的适用场景有什么区别？
- 为什么只读并发访问通常是安全的，但一旦有写入就不安全？
