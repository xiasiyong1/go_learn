# 064. 循环 - 面试追问

## 1. `for condition` 和 `for { if ... break }` 怎么选？

如果循环条件一开始就能表达清楚，用 `for condition`。

```go
for len(queue) > 0 {
	item := queue[0]
	queue = queue[1:]
	handle(item)
}
```

如果退出条件依赖循环内部读取结果，用无限循环加 `break` 更自然。

```go
for {
	line, err := reader.ReadString('\n')
	if err == io.EOF {
		break
	}
	if err != nil {
		return err
	}
	handle(line)
}
```

判断标准是：退出条件放在循环头是否清楚。如果放到循环头反而要引入额外状态变量，就可以使用 `for {}`。

## 2. 为什么 `switch` 里的 `break` 不会跳出外层循环？

因为不带标签的 `break` 退出最近的 `for`、`switch` 或 `select`。如果最近的是 `switch`，它只退出 `switch`。

```go
for _, v := range []int{1, 2, 3} {
	switch v {
	case 2:
		break
	}
	fmt.Println("loop still runs:", v)
}
```

要退出外层循环，用标签：

```go
Loop:
for _, v := range []int{1, 2, 3} {
	switch v {
	case 2:
		break Loop
	}
	fmt.Println(v)
}
```

也可以拆函数，用 `return` 替代标签：

```go
func firstNonOne(values []int) int {
	for _, v := range values {
		switch v {
		case 1:
			continue
		default:
			return v
		}
	}
	return 0
}
```

## 3. 带标签的 `break` 和 `continue` 应该怎么读？

`break Label` 表示退出标签对应的循环；`continue Label` 表示进入标签对应循环的下一轮。

```go
Outer:
for i := 0; i < 3; i++ {
	for j := 0; j < 3; j++ {
		if j == 1 {
			continue Outer
		}
		fmt.Println(i, j)
	}
}
```

这段代码中 `j == 1` 时，会直接进入下一轮 `i`，不会继续当前内层循环。

标签适合少量使用。若标签让读者必须来回跳着读代码，就应该考虑拆函数。

## 4. 循环中启动 goroutine 有什么风险？

风险有两个：数量失控和变量捕获。

数量失控：

```go
for _, job := range jobs {
	go process(job) // jobs 很大时会瞬间创建大量 goroutine
}
```

更稳的方式是限制并发：

```go
sem := make(chan struct{}, 20)

for _, job := range jobs {
	sem <- struct{}{}
	go func(job Job) {
		defer func() { <-sem }()
		process(job)
	}(job)
}
```

变量捕获在新版本 Go 中已经改善了常见 range 场景，但面试时仍应知道原则：传参给 goroutine 最清楚。

```go
for _, job := range jobs {
	go func(job Job) {
		process(job)
	}(job)
}
```

## 5. 遍历时删除元素为什么容易出错？

正向遍历删除会改变后续元素位置，容易跳过元素。

```go
s := []int{1, 2, 2, 3}

for i, v := range s {
	if v == 2 {
		s = append(s[:i], s[i+1:]...)
	}
}

fmt.Println(s) // 可能不是你想要的结果
```

更清晰的过滤写法是原地构造新切片：

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

如果元素里有指针，并且原切片生命周期很长，过滤后还要考虑清理尾部引用，避免对象被底层数组继续持有。
