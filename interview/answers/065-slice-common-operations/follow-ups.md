# 065. 切片操作 - 面试追问

## 1. `append(s[:i], s[i+1:]...)` 到底移动了哪些数据？

它把 `i+1` 到末尾的元素追加到 `s[:i]` 后面，相当于覆盖被删除元素。

```go
s := []int{10, 20, 30, 40}
i := 1

left := s[:i]      // [10]
right := s[i+1:]   // [30 40]
s = append(left, right...)

fmt.Println(s) // [10 30 40]
```

注意这个操作通常仍复用原底层数组，所以原数组尾部可能还残留旧值。

```go
s := []int{10, 20, 30, 40}
s = append(s[:1], s[2:]...)

fmt.Println(s)      // [10 30 40]
fmt.Println(s[:cap(s)]) // 可能看到尾部旧数据
```

业务代码不应该依赖尾部旧数据，但它会影响引用类对象的 GC。

## 2. 删除元素时为什么有时要把尾部置零？

如果切片元素持有指针或引用，被删除元素可能仍留在底层数组的尾部。

```go
type User struct {
	Name string
	Data []byte
}

users := []*User{
	{Name: "a", Data: make([]byte, 1<<20)},
	{Name: "b", Data: make([]byte, 1<<20)},
}

copy(users[0:], users[1:])
users[len(users)-1] = nil // 让旧对象不再被底层数组引用
users = users[:len(users)-1]
```

如果不置零，只要 `users` 的底层数组还活着，旧对象就可能继续可达。

对 `[]int`、`[]float64` 这类不含引用的切片，置零通常只是额外操作，没有 GC 意义。

## 3. 原地过滤 `dst := s[:0]` 会不会破坏原切片？

会。它复用同一个底层数组，把保留元素写回前面。

```go
s := []int{1, 2, 3, 4}
dst := s[:0]

for _, v := range s {
	if v%2 == 0 {
		dst = append(dst, v)
	}
}

fmt.Println(dst) // [2 4]
fmt.Println(s)   // 底层数组前面也已经被改写
```

如果后续还需要原始 `s`，就不要原地过滤。

```go
dst := make([]int, 0, len(s))
for _, v := range s {
	if v%2 == 0 {
		dst = append(dst, v)
	}
}
```

面试回答要说清楚取舍：原地过滤省分配，但破坏原数据。

## 4. 遍历切片时边遍历边删除为什么容易出错？

因为删除会改变后续元素的位置，而 `range` 的迭代过程不会按你的删除逻辑重新规划。

错误示例：

```go
s := []int{1, 2, 2, 3}

for i, v := range s {
	if v == 2 {
		s = append(s[:i], s[i+1:]...)
	}
}

fmt.Println(s) // 结果容易误判
```

推荐写成过滤：

```go
s := []int{1, 2, 2, 3}
dst := s[:0]

for _, v := range s {
	if v != 2 {
		dst = append(dst, v)
	}
}

fmt.Println(dst) // [1 3]
```

如果必须原地按下标删除，可以倒序遍历：

```go
for i := len(s) - 1; i >= 0; i-- {
	if s[i] == 2 {
		s = append(s[:i], s[i+1:]...)
	}
}
```

倒序的好处是删除当前位置不会影响还没遍历的前半部分。

## 5. 插入元素时如何减少分配？

先预估容量。频繁插入时，如果每次都让 `append` 自己扩容，会产生多次分配和复制。

```go
s := make([]int, 0, 100)
s = append(s, 1, 3, 4)
```

插入单个元素：

```go
func insert(s []int, i int, v int) []int {
	s = append(s, 0)
	copy(s[i+1:], s[i:])
	s[i] = v
	return s
}
```

插入多个元素时，先一次性扩出空间，再搬移：

```go
func insertMany(s []int, i int, values ...int) []int {
	oldLen := len(s)
	s = append(s, values...)
	copy(s[i+len(values):], s[i:oldLen])
	copy(s[i:], values)
	return s
}
```

这个写法比多次单个插入更少搬移，也更容易控制容量。
