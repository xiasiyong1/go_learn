# 032. Mutex 和 RWMutex

## 问题

`Mutex` 和 `RWMutex` 有什么区别？如何避免锁误用？

## 核心答案

`Mutex` 是互斥锁，同一时间只允许一个 goroutine 进入临界区。

`RWMutex` 区分读锁和写锁，允许多个读者并发，但写锁需要独占。它适合读多写少且临界区确实能并发读的场景。

锁保护的是共享状态，不是代码块本身。应尽量缩小临界区，避免持锁执行耗时 I/O。

## 代码示例

```go
mu.Lock()
defer mu.Unlock()
```

## 面试追问

- `defer Unlock` 有什么优缺点？
- RWMutex 一定比 Mutex 快吗？
