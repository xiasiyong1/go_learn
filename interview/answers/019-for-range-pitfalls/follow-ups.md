# 019. for range 常见坑 - 面试追问

## 1. range 得到的是元素副本还是原元素？

通常是元素副本。

```go
type User struct {
	Name string
}

users := []User{{Name: "a"}}

for _, u := range users {
	u.Name = "b"
}

fmt.Println(users[0].Name) // a
```

要修改原元素，用索引：

```go
for i := range users {
	users[i].Name = "b"
}
```

如果元素本身是指针，副本也是指向同一对象的指针：

```go
users := []*User{{Name: "a"}}
for _, u := range users {
	u.Name = "b"
}
fmt.Println(users[0].Name) // b
```

## 2. 为什么 `&u` 不等于 `&users[i]`？

`u` 是循环变量，保存的是元素值副本。`&u` 取的是这个循环变量的地址，不是切片元素地址。

```go
users := []User{{Name: "a"}, {Name: "b"}}

var ptrs []*User
for _, u := range users {
	ptrs = append(ptrs, &u)
}

ptrs[0].Name = "x"
fmt.Println(users[0].Name) // a
```

正确写法：

```go
var ptrs []*User
for i := range users {
	ptrs = append(ptrs, &users[i])
}

ptrs[0].Name = "x"
fmt.Println(users[0].Name) // x
```

这道追问的重点不是只背“循环变量复用”，而是要分清“变量副本”和“容器里的元素位置”。

## 3. Go 1.22 之后循环闭包问题是不是完全不用管了？

不是。

Go 1.22 改善的是常见的 `for ... := range ...` 迭代变量捕获问题，每次迭代会有新的变量。但你仍然可能写出共享外部变量的闭包。

安全写法：

```go
for _, task := range tasks {
	go func(t Task) {
		run(t)
	}(task)
}
```

危险写法：

```go
var current Task
for _, task := range tasks {
	current = task
	go func() {
		run(current)
	}()
}
```

这里所有 goroutine 捕获的是同一个外部变量 `current`。

所以面试里可以说：新版本修复了最常见模式，但闭包捕获共享变量这个基本问题仍然要理解。

## 4. range 数组和 range 切片有什么复制差异？

range 数组会复制整个数组：

```go
a := [3]int{1, 2, 3}

for i, v := range a {
	if i == 0 {
		a[1] = 100
	}
	fmt.Println(v)
}
```

遍历值来自数组副本。

range 切片复制的是切片头，底层数组共享：

```go
s := []int{1, 2, 3}

for i, v := range s {
	if i == 0 {
		s[1] = 100
	}
	fmt.Println(v)
}
```

后续迭代可能看到底层数组的修改。

如果数组很大，不想复制，遍历指针或切片：

```go
for _, v := range &a {
	fmt.Println(v)
}
```

## 5. 遍历切片时 append 为什么容易让结果难推理？

range 开始时会确定遍历长度，但 append 可能修改原底层数组，也可能扩容到新数组。

```go
s := []int{1, 2, 3}

for _, v := range s {
	s = append(s, v)
}

fmt.Println(s) // 循环只按初始长度跑 3 次
```

更复杂的是容量足够时，append 会写同一个底层数组；容量不足时，会换新数组。代码可读性会很差。

如果确实要边遍历边追加，写成显式循环：

```go
for i := 0; i < len(s); i++ {
	if needMore(s[i]) {
		s = append(s, next(s[i]))
	}
}
```

这种写法至少把“长度会变化”这件事显式暴露出来。

## 6. map 遍历为什么要排序 key 才能稳定输出？

map 遍历顺序不保证稳定。

```go
for k, v := range m {
	fmt.Println(k, v)
}
```

如果用于日志、测试 golden file、签名、缓存 key，就必须稳定顺序：

```go
keys := make([]string, 0, len(m))
for k := range m {
	keys = append(keys, k)
}
sort.Strings(keys)

for _, k := range keys {
	fmt.Println(k, m[k])
}
```

面试里可以补一句：不要靠“我本机输出顺序固定”判断 map 顺序，语言规范不保证它。
