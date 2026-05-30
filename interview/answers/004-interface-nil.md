# 004. interface nil 陷阱

## 问题

为什么 interface 可能出现“看起来不是 nil，但底层值是 nil”的情况？

## 答案

接口值由两部分组成：动态类型和动态值。只有动态类型和动态值都为 nil 时，接口值才等于 nil。

如果把一个 nil 指针赋给接口变量，接口变量会持有这个指针的动态类型，因此接口本身不是 nil，即使它的动态值是 nil。

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

- `interface{}` 的内部结构是什么？
- 类型断言和类型判断分别适合什么场景？
