# 056. 锁竞争

## 问题

如何定位和优化锁竞争？

## 核心答案

先通过 mutex profile、block profile、trace 或业务指标确认锁竞争位置，不要靠猜。

优化方向包括：缩小临界区；减少持锁 I/O；分片锁；读写分离；使用无共享设计；把全局锁拆成局部锁。

## 代码示例

```go
runtime.SetMutexProfileFraction(1)
```

## 面试追问

- 分片锁的代价是什么？
- RWMutex 读多写少一定更好吗？
