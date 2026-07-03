# 025. 切片内存滞留

## 问题

大切片截取小切片为什么可能导致内存不释放？

## 先给结论

切片是一个描述符，包含指向底层数组的指针、长度和容量。

从大切片截取小切片后，小切片仍然可能指向原来的大底层数组。只要这个小切片还可达，GC 就认为整块底层数组仍然可达，不能回收。

解决办法是：如果只需要长期保存小片段，就复制一份。

```go
small := append([]byte(nil), big[:10]...)
```

## 问题示例

```go
var cache [][]byte

func SaveHeader(data []byte) {
	header := data[:16]
	cache = append(cache, header)
}
```

如果 `data` 是 100 MB，`header` 只有 16 字节，但它仍然引用原来的 100 MB 底层数组。只要 `cache` 持有 `header`，这 100 MB 就可能无法释放。

正确写法：

```go
func SaveHeader(data []byte) {
	header := append([]byte(nil), data[:16]...)
	cache = append(cache, header)
}
```

现在 `header` 指向新的 16 字节数组。原大数组如果没有其他引用，就可以被 GC 回收。

## cap 暴露了底层数组后续空间

```go
data := []byte{1, 2, 3, 4, 5}
part := data[:2]

fmt.Println(len(part), cap(part)) // 2 5
```

`part` 长度是 2，但容量可能是 5，说明它还能看到后面的底层数组空间。

如果调用方 append，可能改到原数组：

```go
part = append(part, 9)
fmt.Println(data) // [1 2 9 4 5]
```

如果返回子切片但不希望调用方 append 污染原数组，可以用完整切片表达式限制容量：

```go
part := data[:2:2]
part = append(part, 9)

fmt.Println(data) // [1 2 3 4 5]
fmt.Println(part) // [1 2 9]
```

但注意：`data[:2:2]` 只限制容量，不释放原数组。要释放大数组，仍然需要复制。

## 复制解除引用

常用写法：

```go
small := append([]byte(nil), big[:n]...)
```

或者：

```go
small := make([]byte, n)
copy(small, big[:n])
```

两者都能让 `small` 指向新的小数组。

什么时候需要复制？

- 从大文件里解析出小字段，并长期缓存。
- 从大 HTTP 响应里保留一个 token。
- 从大 buffer 里截取 key 放到 map。
- 把小片段放进全局变量、队列、缓存、异步任务。

如果小切片只在当前函数短时间使用，不一定需要复制。

## 字符串也有类似问题吗

现代 Go 对字符串/字节转换和子串有不少优化，具体实现也可能随版本演进。面试里更稳的回答是：不要依赖“截取小字符串一定释放大字符串”这种实现细节；如果你从很大的数据中长期保存很小的字符串，并且通过 profile 看到内存滞留，可以显式复制。

```go
small := strings.Clone(big[:10])
```

对 `[]byte` 切片，复制解除底层数组引用这个原则更直接。

## 如何验证

可以用 benchmark 和 heap profile 观察。

```go
func KeepSmallNoCopy(big []byte) []byte {
	return big[:16]
}

func KeepSmallCopy(big []byte) []byte {
	return append([]byte(nil), big[:16]...)
}
```

命令：

```sh
go test -bench . -benchmem
go test -run TestName -memprofile mem.out
go tool pprof mem.out
```

看内存问题时，不只看“在哪里分配”，还要看“为什么对象仍然存活”。

## 面试追问

追问参考答案：[follow-ups.md](follow-ups.md)

- 为什么小切片会让大数组无法释放？
- `s[:n:n]` 能不能释放原数组？
- 什么时候应该复制子切片？
- append 子切片为什么可能污染原数组？
- 如何用 pprof 验证切片内存滞留？
- 字符串截取是否也要考虑类似问题？
