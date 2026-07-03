# 010. make 和 new

## 问题

`make` 和 `new` 有什么区别？

## 先给结论

`new` 分配某类型的零值并返回指针，`make` 初始化 slice、map、channel 这三类运行时结构并返回该类型本身。核心区别是“只分配零值”还是“创建可用的数据结构”。

## 深入理解

### 1. 这道题真正考察什么

- 是否知道 `new(T)` 的结果类型是 `*T`。
- 是否知道 `make` 只用于 slice、map、channel。
- 是否能解释 nil map 不能写、nil channel 会阻塞，所以需要 make。
- 是否能判断 `make([]T, len, cap)` 的 len 和 cap 对后续 append 的影响。

### 2. 底层机制要讲清楚

- `new` 只保证拿到一个被清零的 T 值地址，不负责构造内部运行时结构。
- map 需要初始化哈希表，channel 需要初始化队列和等待队列，slice 可能需要底层数组。
- `make([]T, n)` 会创建长度为 n 的切片，元素已经存在且为零值。
- `make([]T, 0, n)` 创建长度为 0、容量为 n 的切片，更适合逐步 append。

### 3. 工程实践怎么取舍

- 需要指针且零值可用时，用 `new` 或 `&T{}` 都可以，实际更常见是 `&T{}`。
- 创建 map、channel 时用 make，并尽量根据规模给 map 或 slice 容量提示。
- 区分“预先有 n 个元素”和“预计追加 n 个元素”，分别选择 len=n 或 len=0 cap=n。
- 结构体初始化优先用复合字面量表达字段含义。

### 4. 常见误区

- 写 `p := new(map[string]int)` 后直接 `(*p)[k] = v`，仍然会 panic。
- 把 `make([]int, n)` 当成空切片再 append，得到前 n 个零值。
- 给 channel 使用过大的缓冲，掩盖消费慢的问题。
- 把 new 和 make 理解成栈分配与堆分配的区别，实际是否上堆由逃逸分析决定。

## 如何验证理解

- 用 `fmt.Printf("%T")` 看 new 和 make 的返回类型。
- 写测试覆盖 nil map 写入 panic、nil channel select 阻塞等边界。
- 用 `go build -gcflags=-m` 看对象是否逃逸，而不是根据 new/make 猜测。

## 代码示例

```go
p := new(int)        // *int
s := make([]int, 0)  // []int
m := make(map[string]int)
ch := make(chan int)
```

## 面试追问

追问参考答案：[follow-ups.md](follow-ups.md)

- 如果继续追问“make 和 new”的底层机制，应该讲到哪些层次？
- 这个知识点在真实项目里应该如何取舍？
- 最容易踩的坑是什么？为什么这些坑不是背结论就能避免的？
- 如何用测试、工具或 profiling 验证自己的判断？
- 当数据量、并发量或团队规模变大后，这个问题会怎样升级？
