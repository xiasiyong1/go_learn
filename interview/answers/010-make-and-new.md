# 010. make 和 new

## 问题

`make` 和 `new` 有什么区别？

## 核心答案

`new(T)` 分配一个 T 类型的零值并返回 `*T`。

`make` 只用于 slice、map、channel，返回的是初始化后的类型本身，不是指针。它会完成这些运行时数据结构所需的初始化。

## 代码示例

```go
p := new(int)        // *int
s := make([]int, 0)  // []int
m := make(map[string]int)
ch := make(chan int)
```

## 面试追问

- 为什么 `var m map[string]int` 不能直接写入？
- `make([]int, len, cap)` 中 len 和 cap 有什么区别？
