# 003. defer、return 和具名返回值

## 问题

defer 的执行顺序是什么？defer、return 和具名返回值之间是什么关系？

## 答案

`defer` 按后进先出的顺序执行。

函数返回时大致分三步：

1. 给返回值赋值。
2. 执行 `defer`。
3. 真正返回。

如果函数使用具名返回值，`defer` 可以修改这个已经被赋值的返回变量。如果不是具名返回值，`defer` 修改局部变量通常不会影响最终返回结果。

## 代码示例

```go
package main

import "fmt"

func namedReturn() (x int) {
	defer func() {
		x++
	}()
	return 1
}

func unnamedReturn() int {
	x := 1
	defer func() {
		x++
	}()
	return x
}

func main() {
	fmt.Println(namedReturn())   // 2
	fmt.Println(unnamedReturn()) // 1
}
```

## 面试追问

追问参考答案：[follow-ups.md](follow-ups.md)

- `defer` 的参数什么时候求值？
- 在循环中使用 `defer` 有什么风险？
