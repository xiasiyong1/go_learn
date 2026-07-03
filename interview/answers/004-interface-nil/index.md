# 004. interface nil 陷阱

## 问题

为什么 interface 可能出现“看起来不是 nil，但底层值是 nil”的情况？

## 先给结论

interface 的 nil 陷阱来自接口值由“动态类型”和“动态值”两部分组成。只有两者都为空时，接口本身才等于 nil。

## 深入理解

### 1. 这道题真正考察什么

- 是否能区分接口变量本身为 nil 和接口里装了一个 nil 指针。
- 是否知道 `var e error = (*MyErr)(nil)` 时 `e != nil`。
- 是否能解释这个问题为什么常出现在 error 返回值里。
- 是否知道如何设计函数避免把 nil 具体指针装进接口返回。

### 2. 底层机制要讲清楚

- 空接口可以近似理解为 `(type, value)`；非空接口还涉及接口表和方法集。
- 当动态类型是 `*T`、动态值是 nil 时，接口的类型部分非空，所以接口值不等于 nil。
- 接口调用方法时会依据动态类型做分派；如果方法内部解引用 nil 接收者，仍可能 panic。
- 类型断言检查的是接口中的动态类型，不是变量声明时的静态类型。

### 3. 工程实践怎么取舍

- 返回 `error` 时，如果没有错误直接返回字面量 `nil`，不要返回一个类型为 `*MyErr` 的 nil 变量。
- 接口入参里需要判断 nil 时，先判断接口本身，再根据需要用反射或类型断言处理底层 nil。
- 给接口设计方法时，明确 nil receiver 是否是合法状态。
- 尽量让函数返回具体类型或接口之一，不要在同一层来回装箱拆箱。

### 4. 常见误区

- 用 `if err != nil` 判断后发现明明没有错误却进入错误分支。
- 以为接口保存的是“对象本身”，忽略动态类型也参与 nil 判断。
- 用反射判断 nil 时，没有先处理非可 nil 类型导致 panic。
- 在接口里放入 nil slice、nil map、nil func 后误判接口为 nil。

## 如何验证理解

- 写表格测试覆盖 nil 接口、nil 指针装接口、nil slice 装接口等场景。
- 用 `%T` 和 `%#v` 打印动态类型和值，辅助理解接口内容。
- 对 error 返回路径增加测试，确认无错误时返回的确实是 nil 接口。

## 代码示例

```go
package main

import "fmt"

type MyError struct{}

func (e *MyError) Error() string {
	return "my error"
}

func returnsError() error {
	var err *MyError
	return err
}

func main() {
	err := returnsError()
	fmt.Println(err == nil) // false
}
```

## 避免方式

如果没有错误，直接返回接口类型的 nil：

```go
func returnsError() error {
	var err *MyError
	if err == nil {
		return nil
	}
	return err
}
```

## 面试追问

追问参考答案：[follow-ups.md](follow-ups.md)

- 如果继续追问“interface nil 陷阱”的底层机制，应该讲到哪些层次？
- 这个知识点在真实项目里应该如何取舍？
- 最容易踩的坑是什么？为什么这些坑不是背结论就能避免的？
- 如何用测试、工具或 profiling 验证自己的判断？
- 当数据量、并发量或团队规模变大后，这个问题会怎样升级？
