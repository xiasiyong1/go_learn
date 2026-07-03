# 025. 切片内存滞留 - 面试追问

## 1. 为什么小切片会让大数组无法释放？

因为切片头里有指向底层数组的指针。GC 从可达对象出发，只要小切片可达，它指向的底层数组也可达。

```go
var saved []byte

func Save(data []byte) {
	saved = data[:8]
}
```

如果 `data` 是 50 MB，`saved` 只有 8 字节，但仍然可能让 50 MB 底层数组存活。

复制后就断开引用：

```go
func Save(data []byte) {
	saved = append([]byte(nil), data[:8]...)
}
```

## 2. `s[:n:n]` 能不能释放原数组？

不能。它只能限制新切片的容量。

```go
s := []byte{1, 2, 3, 4}
p := s[:2:2]

fmt.Println(len(p), cap(p)) // 2 2
```

这样可以避免 append 写回原数组：

```go
p = append(p, 9)
fmt.Println(s) // [1 2 3 4]
```

但 `p` 仍然指向原数组的前两个元素。只要 `p` 长期存活，原数组仍然可能被保留。

释放大数组要复制：

```go
p = append([]byte(nil), p...)
```

## 3. 什么时候应该复制子切片？

关键看大小和生命周期。

应该复制：

- 原数组很大。
- 子切片很小。
- 子切片会长期保存。
- 原数组没有其他必要引用。

```go
func ExtractToken(resp []byte) []byte {
	token := findToken(resp)
	return append([]byte(nil), token...)
}
```

不一定复制：

```go
func HandleLine(line []byte) {
	field := line[:4]
	useImmediately(field)
}
```

短生命周期、当前函数内立即使用，复制可能只是增加成本。

## 4. append 子切片为什么可能污染原数组？

因为子切片容量可能覆盖原数组后续空间。

```go
s := []int{1, 2, 3, 4}
p := s[:2]

p = append(p, 99)

fmt.Println(s) // [1 2 99 4]
```

如果不希望 append 影响原数组，用完整切片表达式限制容量：

```go
p := s[:2:2]
p = append(p, 99)

fmt.Println(s) // [1 2 3 4]
```

这解决的是 append 污染问题，不是内存释放问题。

## 5. 如何用 pprof 验证切片内存滞留？

思路是构造“保留小切片”的路径，然后看 heap profile 里大数组是否仍然存活。

测试代码可以分两版：

```go
func KeepNoCopy(big []byte) []byte {
	return big[:16]
}

func KeepCopy(big []byte) []byte {
	return append([]byte(nil), big[:16]...)
}
```

命令：

```sh
go test -run TestMemory -memprofile mem.out
go tool pprof mem.out
```

在 pprof 中重点看 `inuse_space`，它表示当前仍然存活的内存。`alloc_space` 是累计分配，更适合找分配热点。

## 6. 字符串截取是否也要考虑类似问题？

思路上要考虑“长期保存小片段是否持有大数据”。具体实现可能随 Go 版本变化，不建议把代码正确性建立在某个优化细节上。

如果 profile 显示大字符串因为小子串长期存活，可以显式复制：

```go
small := strings.Clone(big[:10])
```

对 `[]byte` 来说，复制解除底层数组引用是更常见、更直接的面试重点。
