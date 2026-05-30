# 026. map key 和遍历顺序

## 问题

map 的 key 为什么必须可比较？遍历顺序为什么是随机的？

## 核心答案

map 需要通过 key 的哈希值和相等比较来定位元素，所以 key 必须可比较。slice、map、function 不能作为 map key。

Go 有意不保证 map 遍历顺序，运行时还会引入随机性，防止程序依赖不稳定顺序，也降低某些哈希攻击和实现耦合风险。

## 代码示例

```go
m := map[string]int{"a": 1, "b": 2}
for k, v := range m {
	fmt.Println(k, v)
}
```

## 面试追问

- 如果需要稳定顺序应该怎么做？
- 结构体可以作为 map key 吗？
