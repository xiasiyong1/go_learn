# 007. string、byte 和 rune - 面试追问

## 1. `len(s)` 返回的是什么，为什么中文字符串长度看起来不对？

`len(s)` 返回字节数，不是字符数，也不是 rune 数。

```go
s := "Go语言"

fmt.Println(len(s))         // 8
fmt.Println(len([]rune(s))) // 4
```

原因是 UTF-8 里：

- `G` 占 1 字节。
- `o` 占 1 字节。
- `语` 占 3 字节。
- `言` 占 3 字节。

所以字节数是 `1 + 1 + 3 + 3 = 8`。

如果只是统计 Unicode 码点数，可以用：

```go
func CountRunes(s string) int {
	n := 0
	for range s {
		n++
	}
	return n
}
```

但如果业务需要统计用户感知字符数，rune 数仍然可能不准确。

## 2. `s[i]` 和 `for range s` 的区别是什么？

`s[i]` 按字节取值，类型是 `byte`。`for range s` 按 UTF-8 解码，得到 rune 和它的起始字节下标。

```go
s := "a语"

fmt.Printf("%T %[1]x\n", s[1]) // uint8 e8

for i, r := range s {
	fmt.Printf("i=%d r=%c code=%U\n", i, r, r)
}
```

`语` 的 UTF-8 编码占 3 个字节。`s[1]` 只是第一个字节；`range` 会把这 3 个字节解码成一个 rune。

如果字符串包含非法 UTF-8，`range` 会给出 `utf8.RuneError`：

```go
s := string([]byte{0xff})
for _, r := range s {
	fmt.Printf("%U\n", r) // U+FFFD
}
```

## 3. 按字节截断字符串为什么可能产生乱码？

因为可能把一个多字节 UTF-8 字符截断到一半。

```go
s := "语言"

bad := s[:1]
fmt.Println(utf8.ValidString(bad)) // false
```

按 rune 截断可以保证不会截断单个 UTF-8 码点：

```go
func TruncateRunes(s string, n int) string {
	rs := []rune(s)
	if len(rs) <= n {
		return s
	}
	return string(rs[:n])
}
```

但是这仍然可能截断组合字符或 emoji 序列。所以 UI 显示、昵称、评论等面向用户的文本，要考虑 grapheme cluster，而不只是 rune。

## 4. `rune` 是否等于用户看到的一个字符？

不一定。

`rune` 通常表示 Unicode 码点。用户看到的一个字符可能由多个码点组成。

```go
s := "e\u0301"

fmt.Println(s)              // 看起来可能像 é
fmt.Println(len([]rune(s))) // 2

for _, r := range s {
	fmt.Printf("%c %U\n", r, r)
}
```

emoji 家庭符号也常由多个 rune 组成：

```go
s := "👨‍👩‍👧‍👦"

fmt.Println(len(s))         // 字节数
fmt.Println(len([]rune(s))) // 多个 rune
```

所以：

- 协议长度限制通常按字节。
- Unicode 语义处理可以按 rune。
- 用户可见字符、光标移动、显示宽度、截断要按 grapheme cluster 或显示宽度处理。

## 5. `string` 和 `[]byte` 互转为什么会带来分配？

因为 `string` 不可变，`[]byte` 可变。为了保证修改 `[]byte` 不影响原字符串，转换通常需要复制。

```go
s := "hello"
b := []byte(s)
b[0] = 'H'

fmt.Println(s)         // hello
fmt.Println(string(b)) // Hello
```

热路径里反复转换会增加分配：

```go
func CountErrors(lines [][]byte) int {
	count := 0
	for _, line := range lines {
		if strings.Contains(string(line), "ERROR") {
			count++
		}
	}
	return count
}
```

如果输入本来是 `[]byte`，可以优先用 `bytes` 包：

```go
func CountErrors(lines [][]byte) int {
	count := 0
	needle := []byte("ERROR")
	for _, line := range lines {
		if bytes.Contains(line, needle) {
			count++
		}
	}
	return count
}
```

是否值得优化仍然要看 benchmark，普通业务代码里一次转换未必是问题。

## 6. 处理协议字段、用户昵称、日志拼接时分别应该按什么维度处理？

协议字段通常按 byte，因为协议规定的是字节布局：

```go
if len(payload) > maxBytes {
	return fmt.Errorf("payload too large")
}
```

用户昵称要避免非法 UTF-8，并且不能只按字节随便截断：

```go
func ValidName(name string) bool {
	return utf8.ValidString(name) && len([]rune(name)) <= 20
}
```

如果产品要求“最多 20 个用户可见字符”，这段还不够，需要 grapheme cluster 级别的库。

日志拼接关注分配和可读性，热点路径用 `strings.Builder`：

```go
func BuildLog(user, action string) string {
	var b strings.Builder
	b.Grow(len(user) + len(action) + 16)
	b.WriteString("user=")
	b.WriteString(user)
	b.WriteString(" action=")
	b.WriteString(action)
	return b.String()
}
```

面试回答的重点是：不要把所有字符串问题都归成“字符数”。先判断业务关心的是字节、码点、用户感知字符，还是性能上的分配。
