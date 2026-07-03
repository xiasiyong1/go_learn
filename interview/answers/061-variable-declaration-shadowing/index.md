# 061. 变量

## 问题

Go 中 `var`、短变量声明和变量遮蔽应该怎么理解？

## 先给结论

`var` 用来声明变量，适合包级变量、零值可用、需要显式类型的场景。`:=` 是短变量声明，只能出现在函数内部，并且左边至少要有一个“新变量”。变量遮蔽指内层作用域声明了和外层同名的变量，这在 Go 里合法，但经常造成“我以为改了外层变量，实际只改了内层变量”的 bug。

## 1. `var` 和 `:=` 的使用边界

`var` 可以出现在包级和函数级；`:=` 只能出现在函数内部。

```go
package main

var packageName = "demo" // 包级只能用 var

func main() {
	count := 1 // 函数内部可以用 :=

	var total int // 需要零值或显式类型时，用 var 更清楚
	total += count

	_, _ = packageName, total
}
```

如果想表达“这个变量后面再赋值”或者“这个变量依赖零值可用”，`var` 往往比 `:=` 更清楚。

```go
var err error

if needA {
	err = doA()
} else {
	err = doB()
}

if err != nil {
	return err
}
```

## 2. `:=` 不是赋值，而是声明

短变量声明要求左边至少有一个新变量。

```go
func example() error {
	x := 1
	x = 2  // 赋值：可以
	x := 3 // 编译错误：no new variables on left side of :=

	return nil
}
```

但下面这段可以编译，因为 `n` 是新变量，`err` 是已经存在的变量。

```go
func read() error {
	var err error

	n, err := readSome()
	if err != nil {
		return err
	}

	_ = n
	return nil
}
```

面试里经常追问这一点，是因为它直接关系到你能不能读懂错误处理里的 `:=`。

## 3. 变量遮蔽最容易出现在错误处理里

下面的代码看起来像是给外层 `err` 赋值，实际在 `if` 里面新声明了一个内层 `err`。

```go
func save() error {
	var err error

	if err := writeFile(); err != nil {
		return err
	}

	// 这里的 err 仍然是外层 err，值还是 nil
	return err
}
```

这种写法本身不一定错，因为 `if err := ...; err != nil` 是 Go 里很常见的局部错误处理方式。真正危险的是你以为内层变量会影响外层变量。

更典型的 bug 是事务场景：

```go
func bad(tx Tx) (err error) {
	defer func() {
		if err != nil {
			tx.Rollback()
			return
		}
		tx.Commit()
	}()

	if _, err := tx.Exec("insert"); err != nil {
		return err
	}

	return nil
}
```

这段代码里 `return err` 返回的是内层 `err` 的值，具名返回值最终仍能接到这个错误，所以这个例子不一定出错。真正要警惕的是在内层只记录或修改局部变量，却期待外层 defer 看到变化。

```go
func reallyBad(tx Tx) (err error) {
	defer func() {
		if err != nil {
			tx.Rollback()
		}
	}()

	if _, err := tx.Exec("insert"); err != nil {
		log.Println(err)
		return nil // 外层 err 仍然是 nil，defer 会误以为成功
	}

	return nil
}
```

## 4. 如何避免遮蔽造成误解

第一种方式是缩小变量作用域：如果错误只在当前分支处理，`if err := ...` 很好。

```go
if err := validate(req); err != nil {
	return fmt.Errorf("validate request: %w", err)
}
```

第二种方式是当外层状态需要被后续逻辑读取时，不要用 `:=`，改用明确赋值。

```go
var err error

_, err = tx.Exec("insert")
if err != nil {
	return err
}
```

第三种方式是拆小函数。函数越短，遮蔽越不容易误导读者。

```go
func handle(req Request) error {
	if err := validate(req); err != nil {
		return err
	}
	return save(req)
}
```

## 5. 面试时怎么答

可以按这个顺序回答：

- `var` 和 `:=` 都是声明变量，但 `:=` 只能在函数内部使用。
- `:=` 左边至少要有一个新变量，否则编译失败。
- 内层作用域可以声明同名变量，这叫变量遮蔽。
- 遮蔽不一定是坏事，但在错误处理、事务、defer、循环里很容易误导读者。
- 如果变量后面还要被外层逻辑使用，就用 `var` + `=`，不要用 `:=`。

## 面试追问

追问参考答案：[follow-ups.md](follow-ups.md)

- `:=` 和 `=` 在编译器眼里最大的区别是什么？
- `if err := f(); err != nil` 会不会造成变量遮蔽？
- 变量遮蔽什么时候是好事，什么时候是坏事？
- 为什么事务代码里变量遮蔽特别危险？
- 如何用代码审查或工具发现可疑遮蔽？
