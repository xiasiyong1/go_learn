# 019. for range 常见坑

## 问题

`for range` 有哪些常见坑？

## 先给结论

`for range` 很方便，但要记住几个关键语义：

- range 数组时会复制数组；range 切片时复制的是切片头。
- range 得到的元素值通常是副本，改循环变量不会改原元素。
- map 遍历顺序不稳定，不能依赖。
- 遍历时修改 slice 或 map，要清楚哪些行为有定义、哪些不应依赖。
- Go 1.22 起，`for range` 使用 `:=` 声明的迭代变量按每次迭代创建，但取地址、闭包和外部变量复用仍然要谨慎。

## 修改循环变量不会修改原元素

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

`u` 是元素副本。要修改原切片元素，用索引：

```go
for i := range users {
	users[i].Name = "b"
}
```

如果切片里放的是指针，副本是指针副本，仍然指向原对象：

```go
users := []*User{{Name: "a"}}

for _, u := range users {
	u.Name = "b"
}

fmt.Println(users[0].Name) // b
```

这里修改的是指针指向的对象。

## 对 range 变量取地址要小心

即使在新版本 Go 中，range 变量本身仍然是元素值，不是切片元素位置。

错误示例：

```go
users := []User{{Name: "a"}, {Name: "b"}}

var ptrs []*User
for _, u := range users {
	ptrs = append(ptrs, &u)
}

ptrs[0].Name = "x"
fmt.Println(users[0].Name) // a
```

`&u` 指向的是循环变量，不是 `users[i]`。

正确写法：

```go
var ptrs []*User
for i := range users {
	ptrs = append(ptrs, &users[i])
}
```

如果你想收集值副本，也要写清楚：

```go
var copies []User
for _, u := range users {
	copies = append(copies, u)
}
```

## 循环闭包和 goroutine

Go 1.22 起，常见的 `for _, v := range values` 闭包捕获问题已经改善：每次迭代会有新的 `v`。

但为了让代码在不同写法下都清楚，启动 goroutine 时仍推荐显式传参：

```go
for _, task := range tasks {
	go func(t Task) {
		run(t)
	}(task)
}
```

尤其要避免捕获外部复用变量：

```go
var task Task
for _, t := range tasks {
	task = t
	go func() {
		run(task) // 所有 goroutine 共享外部变量 task
	}()
}
```

这里仍然会有逻辑错误和数据竞争风险。

## range 数组会复制数组

```go
a := [3]int{1, 2, 3}

for i, v := range a {
	if i == 0 {
		a[1] = 100
	}
	fmt.Println(i, v)
}

fmt.Println(a)
```

range 的是数组值时，会复制数组。循环里的 `v` 来自副本，后续修改原数组不一定反映到遍历值里。

如果不想复制大数组，遍历数组指针或切片：

```go
for i, v := range &a {
	fmt.Println(i, v)
}

for i, v := range a[:] {
	fmt.Println(i, v)
}
```

实际项目里更常见的是遍历 slice。

## range 切片复制切片头

range 切片时，复制的是切片头：指针、长度、容量。底层数组仍然共享。

```go
s := []int{1, 2, 3}

for i, v := range s {
	if i == 0 {
		s[1] = 100
	}
	fmt.Println(i, v)
}
```

第二轮的 `v` 会看到底层数组里被改过的值。

但 range 开始时遍历长度已经确定：

```go
s := []int{1, 2, 3}

for _, v := range s {
	s = append(s, v)
}

fmt.Println(s) // 原循环只跑 3 次，不会无限循环
```

遍历时 append 会让代码很难推理，尤其是发生扩容后。需要边遍历边追加时，最好写成显式索引循环，并说明退出条件。

## map 遍历顺序不稳定

```go
m := map[string]int{
	"a": 1,
	"b": 2,
	"c": 3,
}

for k, v := range m {
	fmt.Println(k, v)
}
```

输出顺序不能依赖。写测试或生成稳定输出时，先排序 key：

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

## 遍历 map 时修改

遍历 map 时删除尚未访问或当前 key 是允许的：

```go
for k, v := range m {
	if v == 0 {
		delete(m, k)
	}
}
```

但遍历时新增元素是否会被遍历到，不应该依赖：

```go
for k := range m {
	m[k+"!"] = 1 // 是否遍历到新 key，不要依赖
}
```

并发遍历和写 map 不安全，必须加锁或换并发模型。

## 面试追问

追问参考答案：[follow-ups.md](follow-ups.md)

- range 得到的是元素副本还是原元素？
- 为什么 `&u` 不等于 `&users[i]`？
- Go 1.22 之后循环闭包问题是不是完全不用管了？
- range 数组和 range 切片有什么复制差异？
- 遍历切片时 append 为什么容易让结果难推理？
- map 遍历为什么要排序 key 才能稳定输出？
