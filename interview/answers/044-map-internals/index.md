# 044. map 底层实现

## 问题

Go map 底层大致如何实现？扩容时发生了什么？

## 核心答案

Go map 是哈希表实现。数据分布在 bucket 中，每个 bucket 存储多个 key/value 以及哈希高位信息。冲突过多时会使用 overflow bucket。

当负载因子过高或 overflow bucket 过多时，map 会扩容。扩容不是一次性搬完所有数据，而是在后续读写过程中渐进迁移，减少单次停顿。

## 代码示例

```go
m := make(map[string]int, 1024) // 合理预估容量可减少扩容
```

## 面试追问

追问参考答案：[follow-ups.md](follow-ups.md)

- 为什么 map 取元素返回的是两个值？
- map 中的元素为什么不能直接取地址？
