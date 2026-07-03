# 063. 控制流

## 问题

Go 的 `if`、`switch` 和条件初始化语句有哪些特点？

## 先给结论

Go 的控制流语法比 C/Java 更克制：条件表达式必须是 `bool`，不需要也不能依赖整数真假；`if` 和 `switch` 可以带初始化语句，用来限制临时变量作用域；`switch` 默认不会贯穿到下一个 case，只有显式 `fallthrough` 才会继续执行下一段。

## 1. `if` 条件必须是 bool

Go 不允许把整数、字符串、指针当成真假值。

```go
func main() {
	n := 1

	// if n {
	// 	fmt.Println("ok")
	// }

	if n != 0 {
		fmt.Println("ok")
	}
}
```

这个规则让条件判断更显式。面试中可以补一句：Go 也没有三元表达式，复杂条件更鼓励拆开写。

```go
status := "normal"
if score < 60 {
	status = "failed"
}
```

## 2. `if` 初始化语句能限制变量作用域

`if init; condition {}` 中的变量只在整个 `if/else` 结构内有效。

```go
func handle(path string) error {
	if data, err := os.ReadFile(path); err != nil {
		return err
	} else {
		fmt.Println(len(data))
	}

	// data 和 err 在这里不可见
	return nil
}
```

这适合错误处理，因为错误变量只服务于当前判断。

```go
if err := validate(req); err != nil {
	return fmt.Errorf("validate request: %w", err)
}
```

但如果后续逻辑还要使用变量，就不要把它声明在 `if` 初始化语句里。

```go
data, err := os.ReadFile(path)
if err != nil {
	return err
}

fmt.Println("size:", len(data))
```

## 3. `switch` 默认不会贯穿

Go 的 `switch` 每个 case 执行完会自动结束，不需要写 `break`。

```go
func level(score int) string {
	switch {
	case score >= 90:
		return "A"
	case score >= 80:
		return "B"
	default:
		return "C"
	}
}
```

多个值可以放在同一个 case 中。

```go
switch ext {
case ".jpg", ".jpeg", ".png":
	return "image"
case ".mp4", ".mov":
	return "video"
default:
	return "unknown"
}
```

## 4. `fallthrough` 很少用，而且不重新判断下一个 case

`fallthrough` 会直接执行下一个 case 的语句，不会检查下一个 case 的条件。

```go
func demo(n int) {
	switch n {
	case 1:
		fmt.Println("one")
		fallthrough
	case 100:
		fmt.Println("hundred or fallthrough")
	}
}

demo(1)
// 输出：
// one
// hundred or fallthrough
```

这就是它危险的地方：读者容易误以为下一个 case 条件也成立。真实项目里通常用多个 case 值、函数提取或显式调用来替代 `fallthrough`。

## 5. `switch` 也可以带初始化语句

这在需要先计算一个局部值再分支时很有用。

```go
switch ext := strings.ToLower(filepath.Ext(name)); ext {
case ".go":
	return "go source"
case ".md":
	return "markdown"
default:
	return "other"
}
```

`ext` 只在这个 `switch` 内部可见，不会泄露到后续代码。

## 6. 面试时怎么答

可以这样回答：

- `if` 条件必须是 bool，不能像 C 一样用 `0/1` 表示真假。
- `if` 和 `switch` 都可以带初始化语句，适合限制临时变量作用域。
- `switch` 默认 break，不会自动贯穿。
- `fallthrough` 会直接执行下一个 case，不会重新判断条件，所以真实项目里很少用。
- 当分支变复杂时，应该用命名变量、拆函数或表驱动降低复杂度。

## 面试追问

追问参考答案：[follow-ups.md](follow-ups.md)

- 为什么 Go 不允许 `if 1 {}`？
- `if err := f(); err != nil` 的变量作用域到哪里？
- Go 的 `switch` 和 C/Java 的 switch 最大区别是什么？
- `fallthrough` 为什么容易误导？
- 分支很多时，什么时候继续用 switch，什么时候改成表驱动？
