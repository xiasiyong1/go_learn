# 064. 循环

## 问题

Go 为什么只有 `for` 循环？`break`、`continue` 和标签怎么用？

## 先给结论

Go 只有一个循环关键字 `for`，但它能表达三类场景：传统计数循环、条件循环和无限循环。`break` 默认跳出最近的 `for`、`switch` 或 `select`，`continue` 默认进入最近一层 `for` 的下一轮。标签可以控制外层循环，但应该谨慎使用。

## 1. 三种常见 `for` 写法

传统计数循环：

```go
for i := 0; i < 3; i++ {
	fmt.Println(i)
}
```

条件循环，相当于其他语言里的 `while`：

```go
n := 3
for n > 0 {
	fmt.Println(n)
	n--
}
```

无限循环，常用于服务主循环、重试、消费队列：

```go
for {
	msg, ok := receive()
	if !ok {
		break
	}
	handle(msg)
}
```

无限循环必须有清晰退出路径，否则很容易变成 CPU 空转或 goroutine 泄漏。

## 2. `break` 只跳出最近的一层结构

在循环里套 `switch` 时，`break` 很容易被误解。

```go
for _, v := range []int{1, 2, 3} {
	switch v {
	case 2:
		break // 只退出 switch，不退出 for
	}
	fmt.Println(v)
}
```

这段代码仍然会继续下一轮 `for`。如果想跳出外层循环，需要标签。

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

标签名应该表达意图，不要随便写成 `A`、`B`。

## 3. `continue` 只作用于循环

`continue` 会跳过当前循环剩余语句，进入下一轮。

```go
for _, v := range []int{1, 2, 3, 4} {
	if v%2 != 0 {
		continue
	}
	fmt.Println(v) // 2, 4
}
```

带标签的 `continue` 可以继续外层循环。

```go
Outer:
for i := 0; i < 3; i++ {
	for j := 0; j < 3; j++ {
		if i == j {
			continue Outer
		}
		fmt.Println(i, j)
	}
}
```

这种写法能减少状态变量，但嵌套太深时通常应该拆函数。

## 4. 标签不是 goto 的替代品

标签适合解决多层循环的“提前结束”问题。

```go
found := false

Search:
for i, row := range matrix {
	for j, v := range row {
		if v == target {
			fmt.Println("found at", i, j)
			found = true
			break Search
		}
	}
}

_ = found
```

但如果标签很多，说明控制流已经复杂。更清楚的方式是提取函数，用 `return` 表达结束。

```go
func find(matrix [][]int, target int) (int, int, bool) {
	for i, row := range matrix {
		for j, v := range row {
			if v == target {
				return i, j, true
			}
		}
	}
	return 0, 0, false
}
```

## 5. 循环里要警惕资源和阻塞

初学者容易只关注语法，忽略循环体里做了什么。循环里的分配、锁、I/O、goroutine 启动都会被次数放大。

```go
for _, url := range urls {
	go fetch(url) // 如果 urls 很大，会瞬间启动大量 goroutine
}
```

更稳的做法是限制并发：

```go
sem := make(chan struct{}, 10)

for _, url := range urls {
	sem <- struct{}{}
	go func(url string) {
		defer func() { <-sem }()
		fetch(url)
	}(url)
}
```

这不是 `for` 语法本身的问题，但面试里能主动说出来，会体现工程意识。

## 6. 面试时怎么答

可以这样组织：

- Go 只有 `for`，但能表达计数循环、条件循环和无限循环。
- `break` 默认退出最近的 `for/switch/select`，在 `switch` 嵌套循环时尤其容易误判。
- `continue` 只用于循环，默认继续最近一层循环。
- 标签可以控制外层循环，但只适合少量、多层循环提前退出场景。
- 复杂嵌套更推荐拆函数，用 `return` 让控制流变简单。

## 面试追问

追问参考答案：[follow-ups.md](follow-ups.md)

- `for condition` 和 `for { if ... break }` 怎么选？
- 为什么 `switch` 里的 `break` 不会跳出外层循环？
- 带标签的 `break` 和 `continue` 应该怎么读？
- 循环中启动 goroutine 有什么风险？
- 遍历时删除元素为什么容易出错？
