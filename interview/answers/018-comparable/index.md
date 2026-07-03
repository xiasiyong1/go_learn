# 018. 可比较类型

## 问题

Go 中哪些类型可以比较？`comparable` 表示什么？

## 先给结论

能用 `==` 和 `!=` 的类型叫可比较类型。map 的 key 必须是可比较类型。

常见规则：

- 布尔、数字、字符串、指针、channel 可以比较。
- 数组是否可比较，取决于元素是否可比较。
- 结构体是否可比较，取决于所有字段是否可比较。
- slice、map、func 不能相互比较，只能和 `nil` 比较。
- interface 静态上可以比较，但如果动态值不可比较，运行时会 panic。
- 泛型里的 `comparable` 是类型约束，表示类型参数支持 `==` 和 `!=`。

## 基础类型可以比较

```go
fmt.Println(1 == 1)
fmt.Println("go" == "go")
fmt.Println(true != false)
```

指针比较的是地址：

```go
a := User{Name: "Tom"}
b := User{Name: "Tom"}

fmt.Println(&a == &b) // false
```

这里内容一样，但地址不同。不要把指针相等误认为内容相等。

channel 也可以比较：

```go
ch1 := make(chan int)
ch2 := ch1

fmt.Println(ch1 == ch2) // true
```

比较的是它们是否引用同一个 channel。

## slice、map、func 不可比较

```go
a := []int{1, 2}
b := []int{1, 2}

// fmt.Println(a == b) // 编译错误
```

slice 可以和 `nil` 比较：

```go
var s []int
fmt.Println(s == nil) // true
```

map 和 func 也是：

```go
var m map[string]int
var f func()

fmt.Println(m == nil) // true
fmt.Println(f == nil) // true
```

如果要比较 slice 内容，用 `slices.Equal`：

```go
fmt.Println(slices.Equal([]int{1, 2}, []int{1, 2})) // true
```

如果要比较 map 内容，用 `maps.Equal` 或自定义比较：

```go
fmt.Println(maps.Equal(
	map[string]int{"a": 1},
	map[string]int{"a": 1},
)) // true
```

## 数组和结构体取决于内部元素

数组元素可比较，数组就可比较：

```go
fmt.Println([2]int{1, 2} == [2]int{1, 2}) // true
```

数组元素不可比较，数组也不可比较：

```go
// _ = [1][]int{} == [1][]int{} // 编译错误
```

结构体同理：

```go
type Point struct {
	X int
	Y int
}

fmt.Println(Point{1, 2} == Point{1, 2}) // true
```

如果结构体里加了 slice 字段，它就不可比较：

```go
type User struct {
	ID   int64
	Tags []string
}

// _ = User{} == User{} // 编译错误
```

这也是公共类型设计里的兼容性风险：一个原本可作为 map key 的结构体，新增不可比较字段后，调用方代码可能直接编译失败。

## map key 为什么必须可比较

map key 需要两个能力：

- 能计算稳定哈希。
- 能判断两个 key 是否相等。

```go
type Key struct {
	TenantID int64
	UserID   int64
}

visits := map[Key]int{}
visits[Key{TenantID: 1, UserID: 2}]++
```

这个结构体所有字段都可比较，所以能做 key。

如果 key 里有 slice，就不能做 map key：

```go
type BadKey struct {
	Parts []string
}

// m := map[BadKey]int{} // 编译错误
```

可以改成稳定字符串或固定数组：

```go
type Key struct {
	Path string
}

key := Key{Path: strings.Join(parts, "/")}
```

具体选择要看是否可能出现分隔符冲突、是否需要性能优化。

## interface 比较的运行时 panic

接口值可以比较，但比较时会看动态值。

```go
var a any = 1
var b any = 1
fmt.Println(a == b) // true
```

如果接口里装的是不可比较类型，比较会 panic：

```go
var a any = []int{1}
var b any = []int{1}

fmt.Println(a == b) // panic: comparing uncomparable type []int
```

这也是 `map[any]V` 的风险：

```go
m := map[any]int{}
m["go"] = 1
m[10] = 2

// m[[]int{1}] = 3 // panic: hash of unhashable type []int
```

如果 key 类型来自外部输入，不要随便用 `any` 做 map key。

## 泛型里的 comparable

泛型函数里如果要使用 `==`，类型参数必须被约束为可比较。

```go
func Index[T comparable](items []T, target T) int {
	for i, item := range items {
		if item == target {
			return i
		}
	}
	return -1
}
```

如果没有 `comparable`：

```go
func BadIndex[T any](items []T, target T) int {
	for i, item := range items {
		// if item == target { // 编译错误
		// 	return i
		// }
		_ = item
	}
	return -1
}
```

`comparable` 只说明可以用 `==`，不说明业务上这个比较一定符合你的相等语义。例如指针可比较，但比较的是地址，不是内容。

## 面试追问

追问参考答案：[follow-ups.md](follow-ups.md)

- 哪些类型不能用 `==` 比较？它们能和 `nil` 比较吗？
- 数组和结构体什么时候可以比较？
- 为什么 map key 必须可比较？
- interface 比较为什么可能运行时 panic？
- 指针相等和内容相等有什么区别？
- 泛型里的 `comparable` 解决了什么问题，又不解决什么问题？
