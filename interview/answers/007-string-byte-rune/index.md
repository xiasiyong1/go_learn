# 007. string、byte 和 rune

## 问题

`string`、`byte` 和 `rune` 有什么区别？

## 先给结论

- `string` 是只读字节序列。
- `byte` 是 `uint8` 的别名，通常表示一个原始字节。
- `rune` 是 `int32` 的别名，通常表示一个 Unicode 码点。

难点在于：字节、码点、字符、用户看到的一个“字”，不是同一个概念。

```go
s := "Go语言"

fmt.Println(len(s)) // 8，返回字节数

for i, r := range s {
	fmt.Printf("index=%d rune=%c code=%U\n", i, r, r)
}
```

`Go` 各占 1 个字节，`语` 和 `言` 在 UTF-8 中各占 3 个字节，所以总长度是 8。

## string 是只读字节序列

`string` 本身不保证内容一定是合法 UTF-8。它只是一段不可变的字节。

```go
s := string([]byte{0xff, 0xfe, 0xfd})
fmt.Println(len(s))        // 3
fmt.Printf("% x\n", s)     // ff fe fd
fmt.Println(utf8.ValidString(s)) // false
```

Go 源码里的字符串字面量通常是 UTF-8，但从文件、网络、二进制协议读到的字符串不一定是合法 UTF-8。

## 索引 string 得到 byte

```go
s := "语言"

fmt.Println(len(s)) // 6
fmt.Printf("%x\n", s[0])
fmt.Printf("%T\n", s[0]) // uint8
```

`s[0]` 得到的是第一个字节，不是第一个中文字符。

如果按字节截断，就可能破坏 UTF-8 编码：

```go
s := "语言"
bad := s[:1]

fmt.Println(bad)
fmt.Println(utf8.ValidString(bad)) // false
```

所以处理二进制协议、哈希、加密、文件格式时按 byte 是对的；处理文本字符时，不能简单按 byte 切。

## range string 得到 rune

`for range` 遍历字符串时，会按 UTF-8 解码，每次得到：

- `i`：当前 rune 起始的字节下标。
- `r`：解码出的 rune。

```go
s := "a语b"

for i, r := range s {
	fmt.Printf("i=%d r=%c bytes=%d\n", i, r, utf8.RuneLen(r))
}
```

输出里的下标会是 `0, 1, 4`，因为 `语` 占 3 个字节。

如果字符串里有非法 UTF-8，`range` 会产生 `utf8.RuneError`：

```go
s := string([]byte{'a', 0xff, 'b'})

for i, r := range s {
	fmt.Printf("i=%d r=%U\n", i, r)
}
```

非法字节会被解码成 `U+FFFD`。

## []byte 和 []rune 的选择

按字节处理：

```go
func HasPrefixMagic(data []byte) bool {
	return len(data) >= 2 && data[0] == 0xca && data[1] == 0xfe
}
```

按 Unicode 码点处理：

```go
func RuneCount(s string) int {
	count := 0
	for range s {
		count++
	}
	return count
}
```

也可以转成 `[]rune` 做按码点截断：

```go
func TruncateRunes(s string, n int) string {
	rs := []rune(s)
	if len(rs) <= n {
		return s
	}
	return string(rs[:n])
}
```

但要注意：按 rune 截断仍然不等于按用户感知字符截断。

## rune 不等于用户看到的一个字符

有些用户看到的一个字符可能由多个 rune 组成。

```go
s := "e\u0301" // e + 组合音调，看起来像 é

fmt.Println(len(s)) // 3 字节

for i, r := range s {
	fmt.Printf("i=%d r=%c code=%U\n", i, r, r)
}
```

这里用户看起来可能是一个字符，但它包含两个 rune：`e` 和组合音调。

emoji 也经常由多个码点组成：

```go
s := "👨‍👩‍👧‍👦"

fmt.Println(len(s))         // 字节数
fmt.Println(len([]rune(s))) // rune 数，不是用户感知字符数
```

如果业务要按“用户看到的字符”截断、计算显示宽度、处理 emoji 序列，应使用专门库处理 Unicode grapheme cluster。标准库的 `len`、`range`、`[]rune` 解决的是字节和码点层面的问题。

## string 和 []byte 转换通常会复制

```go
s := "hello"
b := []byte(s)
b[0] = 'H'

fmt.Println(s)         // hello
fmt.Println(string(b)) // Hello
```

因为 `string` 不可变，转成 `[]byte` 后修改不能影响原字符串。反过来 `string(b)` 通常也会复制，避免后续修改 `b` 影响字符串。

在热路径频繁转换会增加分配：

```go
func Bad(lines [][]byte) int {
	n := 0
	for _, line := range lines {
		if strings.HasPrefix(string(line), "ERROR") {
			n++
		}
	}
	return n
}
```

如果数据本来就是 `[]byte`，优先用 `bytes` 包：

```go
func Good(lines [][]byte) int {
	n := 0
	for _, line := range lines {
		if bytes.HasPrefix(line, []byte("ERROR")) {
			n++
		}
	}
	return n
}
```

## 大量拼接用 strings.Builder

错误示例：

```go
func Join(items []string) string {
	s := ""
	for _, item := range items {
		s += item
	}
	return s
}
```

更好的写法：

```go
func Join(items []string) string {
	var b strings.Builder
	for _, item := range items {
		b.WriteString(item)
	}
	return b.String()
}
```

如果能估计大小，可以 `Grow`：

```go
func Join(items []string) string {
	var size int
	for _, item := range items {
		size += len(item)
	}

	var b strings.Builder
	b.Grow(size)
	for _, item := range items {
		b.WriteString(item)
	}
	return b.String()
}
```

## 面试追问

追问参考答案：[follow-ups.md](follow-ups.md)

- `len(s)` 返回的是什么，为什么中文字符串长度看起来不对？
- `s[i]` 和 `for range s` 的区别是什么？
- 按字节截断字符串为什么可能产生乱码？
- `rune` 是否等于用户看到的一个字符？
- `string` 和 `[]byte` 互转为什么会带来分配？
- 处理协议字段、用户昵称、日志拼接时分别应该按什么维度处理？
