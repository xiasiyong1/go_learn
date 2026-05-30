# 051. sync.Pool - 面试追问

## 追问与参考答案

### 1. Put 回 Pool 前为什么通常要 Reset？

对象放回 Pool 前通常要 Reset，避免下一个使用者读到上一次的脏数据，也避免大 buffer 长期保留过大的底层数组。Reset 是复用对象前恢复干净状态的关键步骤。

### 2. sync.Pool 在高并发下为什么性能较好？

`sync.Pool` 有 per-P 本地缓存，常见 Get/Put 可以减少全局锁竞争。GC 又可以清理池内对象，避免 Pool 成为永久持有内存的结构。
