# 007. string、byte 和 rune

## 问题

`string`、`byte` 和 `rune` 有什么区别？

## 核心答案

`string` 是只读字节序列，不等于字符数组。Go 源码通常使用 UTF-8 编码，所以一个中文字符会占多个字节。

`byte` 是 `uint8` 的别名，表示一个字节。`rune` 是 `int32` 的别名，通常表示一个 Unicode 码点。

按下标访问字符串拿到的是字节，使用 `for range` 遍历字符串时拿到的是 rune。

## 代码示例

```go
s := "Go语言"

fmt.Println(len(s)) // 字节数，不是字符数

for i, r := range s {
	fmt.Println(i, r, string(r))
}
```

## 面试追问

追问参考答案：[follow-ups.md](follow-ups.md)

- 如何正确统计字符串中的字符数量？
- `[]byte(s)` 和 `string(b)` 是否会复制内存？
