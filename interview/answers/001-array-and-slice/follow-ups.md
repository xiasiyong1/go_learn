# 001. 数组和切片 - 面试追问

## 1. 为什么 `[3]int` 和 `[4]int` 是不同类型？

因为数组长度是数组类型的一部分。

```go
var a [3]int
var b [4]int

// a = b // cannot use b as [3]int
```

这个设计让数组适合表达固定长度数据，例如 MD5 是 `[16]byte`，SHA-256 是 `[32]byte`。长度不同就不应该混用。

```go
func useMD5(sum [16]byte) {}
func useSHA256(sum [32]byte) {}
```

如果长度不固定，通常用切片 `[]T`。

## 2. 切片传参后，函数里修改元素和 append 的效果为什么不同？

修改元素会通过切片头里的指针写到底层数组。

```go
func setFirst(s []int) {
	s[0] = 100
}

s := []int{1, 2, 3}
setFirst(s)
fmt.Println(s) // [100 2 3]
```

append 修改的是函数内那份切片头。如果没有把新切片返回给调用方，调用方的 len 不会变。

```go
func add(s []int) {
	s = append(s, 4)
}

s := []int{1, 2, 3}
add(s)
fmt.Println(s) // [1 2 3]
```

正确写法是返回新切片。

```go
func add(s []int) []int {
	return append(s, 4)
}
```

## 3. `a[:2]` 和 `a[:2:2]` 有什么区别？

`a[:2]` 只设置长度，不限制容量。

```go
a := []int{1, 2, 3}
b := a[:2]
fmt.Println(len(b), cap(b)) // 2 3
```

`a[:2:2]` 同时设置长度和容量。

```go
b := a[:2:2]
fmt.Println(len(b), cap(b)) // 2 2
```

限制容量后，append 会更容易分配新数组，避免覆盖原数组后面的元素。

```go
a := []int{1, 2, 3}
b := a[:2:2]
b = append(b, 100)

fmt.Println(a) // [1 2 3]
fmt.Println(b) // [1 2 100]
```

## 4. 为什么小切片会让大数组无法被 GC？

GC 看的是对象是否仍然可达。小切片的指针仍然指向大数组的一部分，所以整个底层数组都可达。

```go
func leak() []byte {
	buf := make([]byte, 100<<20)
	return buf[:1]
}
```

如果返回值长期存活，100 MiB 的底层数组就会长期存活。

复制即可解除引用。

```go
func noLeak() []byte {
	buf := make([]byte, 100<<20)
	out := make([]byte, 1)
	copy(out, buf[:1])
	return out
}
```

完整切片表达式只能限制 cap，不能让原大数组被释放；要释放必须复制。

## 5. 如何判断应该复制切片还是共享底层数组？

共享适合短生命周期、只读、明确所有权的场景。

```go
func sum(s []int) int {
	total := 0
	for _, v := range s {
		total += v
	}
	return total
}
```

复制适合这些场景：

- 要长期保存子切片。
- 不希望调用方后续修改影响自己。
- 要把数据交给异步 goroutine。
- 要缓存来自复用 buffer 的数据。

```go
func keep(input []byte) []byte {
	return append([]byte(nil), input...)
}
```

判断标准不是“复制一定好”或“共享一定快”，而是所有权和生命周期是否清楚。
