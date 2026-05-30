# 010. make 和 new - 面试追问

## 追问与参考答案

### 1. 为什么 `var m map[string]int` 不能直接写入？

`var m map[string]int` 得到的是 nil map，没有底层哈希表，读取可以，写入会 panic。必须用 `make(map[string]int)` 初始化后才能写入。

### 2. `make([]int, len, cap)` 中 len 和 cap 有什么区别？

`len` 是当前可访问元素数量，`cap` 是底层数组从起点开始还能容纳的元素数量。append 超过 cap 时会重新分配底层数组并复制元素。
