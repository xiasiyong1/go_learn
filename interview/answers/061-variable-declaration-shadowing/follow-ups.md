# 061. 变量 - 面试追问

## 1. `:=` 和 `=` 在编译器眼里最大的区别是什么？

`:=` 是声明语句，`=` 是赋值语句。声明会创建新变量，赋值只修改已有变量。

```go
func main() {
	x := 1 // 声明 x
	x = 2  // 修改 x

	// x := 3
	// 编译失败：左边没有新变量
}
```

短变量声明允许“旧变量 + 新变量”混用：

```go
func f() error {
	err := step1()
	if err != nil {
		return err
	}

	n, err := step2() // n 是新变量，所以这一行合法；err 被重新赋值
	_, _ = n, err
	return nil
}
```

追问点在于：你看到 `:=` 时，不能只读成“赋值”，要判断左边哪些名字是新声明的。

## 2. `if err := f(); err != nil` 会不会造成变量遮蔽？

可能会。它一定会创建一个只在 `if/else` 语句内部可见的 `err`。如果外层也有 `err`，那就是遮蔽。

```go
func demo() error {
	var err error

	if err := do(); err != nil {
		return err // 这里是内层 err
	}

	return err // 这里是外层 err
}
```

这不一定是坏写法。局部错误只在局部处理时，这种写法很清楚：

```go
if err := validate(input); err != nil {
	return fmt.Errorf("invalid input: %w", err)
}
```

坏的是你后面还想用外层 `err` 判断状态，却在前面用 `:=` 创建了内层 `err`。

## 3. 变量遮蔽什么时候是好事，什么时候是坏事？

好处是可以把变量限制在很小的作用域里，避免污染后续代码。

```go
if user, err := loadUser(id); err == nil {
	fmt.Println(user.Name)
}
// user 和 err 在这里都不可见
```

坏处是同名变量会让读者误判代码修改的是谁。

```go
func bad() {
	count := 0

	for _, item := range []int{1, 2, 3} {
		if item > 1 {
			count := count + 1 // 新 count，外层 count 没变
			_ = count
		}
	}

	fmt.Println(count) // 0
}
```

这类代码最好写成赋值：

```go
count = count + 1
```

## 4. 为什么事务代码里变量遮蔽特别危险？

事务代码常用 defer 根据错误决定提交还是回滚。如果 defer 看的是外层 `err`，但中间创建了内层 `err`，就可能让 defer 看到错误状态不准确。

错误示例：

```go
func save(tx Tx) (err error) {
	defer func() {
		if err != nil {
			tx.Rollback()
			return
		}
		tx.Commit()
	}()

	if _, err := tx.Exec("insert"); err != nil {
		log.Println("insert failed:", err)
		return nil // 外层 err 还是 nil，最后会 Commit
	}

	return nil
}
```

更稳的写法是不要让事务状态依赖容易被遮蔽的变量：

```go
func save(tx Tx) error {
	if _, err := tx.Exec("insert"); err != nil {
		tx.Rollback()
		return err
	}

	return tx.Commit()
}
```

如果必须 defer，也要确保赋值的是具名返回值或外层变量。

## 5. 如何用工具发现可疑遮蔽？

标准 `go vet` 不会默认检查所有 shadow 问题。团队通常会引入静态检查工具，或者在 code review 里重点看这些位置：

- `:=` 出现在已有 `err` 的函数里。
- `:=` 出现在事务、defer、循环内。
- 内层变量名和外层状态变量同名，例如 `err`、`ctx`、`cancel`、`tx`。

最小自检方式是打印地址：

```go
func main() {
	x := 1
	fmt.Printf("outer: %p\n", &x)

	{
		x := 2
		fmt.Printf("inner: %p\n", &x)
	}
}
```

两个地址不同，说明它们是两个变量。
