# 063. 控制流 - 面试追问

## 1. 为什么 Go 不允许 `if 1 {}`？

Go 要求条件表达式必须是 `bool`，这样可以避免“哪些值算真、哪些值算假”的隐式规则。

```go
func main() {
	n := 1

	// if n {             // 编译失败
	// 	fmt.Println(n)
	// }

	if n > 0 {
		fmt.Println(n)
	}
}
```

这条规则也影响指针、字符串和 slice：

```go
var s []int

// if s { } // 编译失败

if s != nil {
	fmt.Println("not nil")
}
```

面试回答可以强调：Go 倾向显式判断，减少跨语言习惯带来的歧义。

## 2. `if err := f(); err != nil` 的变量作用域到哪里？

初始化语句里的变量在整个 `if/else` 结构中有效，离开后不可见。

```go
if user, err := loadUser(id); err != nil {
	return err
} else {
	fmt.Println(user.Name) // user 和 err 在 else 里也可见
}

// user 在这里不可见
```

如果后面还要用 `user`，就不要把它声明在 `if` 里。

```go
user, err := loadUser(id)
if err != nil {
	return err
}

return render(user)
```

这也是 Go 代码常见风格：临时变量尽量小作用域，需要跨步骤使用的变量就放到外面。

## 3. Go 的 `switch` 和 C/Java 的 switch 最大区别是什么？

Go 默认不会贯穿，所以不需要每个 case 都写 `break`。

```go
switch status {
case "new":
	return "created"
case "done":
	return "finished"
default:
	return "unknown"
}
```

在 C/Java 里忘记 `break` 是常见 bug；在 Go 里只有显式写 `fallthrough` 才会继续执行下一个 case。

Go 的 switch 也更灵活，可以没有 switch 表达式：

```go
switch {
case age < 18:
	return "child"
case age < 60:
	return "adult"
default:
	return "senior"
}
```

这相当于一组更清晰的 if-else。

## 4. `fallthrough` 为什么容易误导？

因为它不会判断下一个 case 条件，直接进入下一个 case 代码块。

```go
func print(n int) {
	switch n {
	case 1:
		fmt.Println("one")
		fallthrough
	case 2:
		fmt.Println("two")
	}
}

print(1)
// one
// two
```

`n` 明明不是 2，却执行了 case 2 的代码。更清楚的写法通常是把共同逻辑提取出来：

```go
switch n {
case 1:
	fmt.Println("one")
	printCommon()
case 2:
	fmt.Println("two")
	printCommon()
}
```

如果只是多个值走同一个分支，用逗号列出来：

```go
switch code {
case 200, 201, 204:
	return "success"
}
```

## 5. 分支很多时，什么时候继续用 switch，什么时候改成表驱动？

如果分支逻辑短、数量少、每个 case 都有不同处理，`switch` 很清楚。

```go
switch op {
case "add":
	return a + b
case "sub":
	return a - b
default:
	return 0
}
```

如果分支只是“根据 key 查一个配置、函数或固定结果”，表驱动更好。

```go
var handlers = map[string]func(int, int) int{
	"add": func(a, b int) int { return a + b },
	"sub": func(a, b int) int { return a - b },
}

fn, ok := handlers[op]
if !ok {
	return 0
}
return fn(a, b)
```

判断标准是：分支数量增长时，修改是否仍然局部、可测试、可读。如果每加一种类型都要改很长的 switch，通常说明可以考虑表驱动或接口多态。
