# 007. string、byte 和 rune

## 问题

`string`、`byte` 和 `rune` 有什么区别？

## 先给结论

`string` 是只读字节序列，`byte` 是 `uint8`，`rune` 是 `int32`，通常表示一个 Unicode 码点。难点在于字符、字节、码点和用户感知字符不是同一个概念。

## 深入理解

### 1. 这道题真正考察什么

- 是否知道 `len(s)` 返回字节数，不是字符数。
- 是否能解释 `range string` 按 UTF-8 解码出 rune。
- 是否知道 `[]byte(s)` 和 `string(b)` 通常会复制。
- 是否能处理中文、emoji、组合字符等边界。

### 2. 底层机制要讲清楚

- Go 源码字符串字面量通常是 UTF-8，但 string 本身只保证是不可变字节序列。
- 索引字符串得到的是 byte；`for range` 得到的是 rune 起始字节位置和 rune 值。
- 一个 rune 可能占 1 到 4 个 UTF-8 字节。
- 用户看到的一个字符可能由多个 rune 组成，例如组合音标或部分 emoji 序列。

### 3. 工程实践怎么取舍

- 处理网络协议、文件格式、哈希时按 byte 处理。
- 处理 Unicode 码点时用 `[]rune` 或 `range`。
- 处理用户感知字符、宽度和截断时，使用专门库，不要只按 rune 数截断。
- 大量字符串拼接用 `strings.Builder`，避免反复创建临时字符串。

### 4. 常见误区

- 用 `len` 统计中文字符数。
- 按字节截断 UTF-8 字符串，导致非法编码。
- 认为 rune 等于用户看到的字符。
- 在热路径频繁 `string` 和 `[]byte` 互转，制造额外分配。

## 如何验证理解

- 用 `%x` 打印字节，用 `%U` 打印 rune，观察 UTF-8 编码。
- 写测试覆盖 ASCII、中文、emoji、组合字符。
- 用 benchmark 比较字符串拼接、Builder 和 byte buffer 的分配。

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

- 如果继续追问“string、byte 和 rune”的底层机制，应该讲到哪些层次？
- 这个知识点在真实项目里应该如何取舍？
- 最容易踩的坑是什么？为什么这些坑不是背结论就能避免的？
- 如何用测试、工具或 profiling 验证自己的判断？
- 当数据量、并发量或团队规模变大后，这个问题会怎样升级？
