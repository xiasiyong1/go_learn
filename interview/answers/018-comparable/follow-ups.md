# 018. 可比较类型 - 面试追问

## 1. 哪些类型不能用 `==` 比较？它们能和 `nil` 比较吗？

slice、map、func 不能彼此比较。

```go
a := []int{1}
b := []int{1}

// fmt.Println(a == b) // 编译错误
```

但它们可以和 `nil` 比较：

```go
var s []int
var m map[string]int
var f func()

fmt.Println(s == nil) // true
fmt.Println(m == nil) // true
fmt.Println(f == nil) // true
```

如果要比较内容，选择对应工具：

```go
slices.Equal([]int{1}, []int{1})
maps.Equal(map[string]int{"a": 1}, map[string]int{"a": 1})
```

func 通常没有通用内容相等语义，只能判断是否为 nil。

## 2. 数组和结构体什么时候可以比较？

数组元素可比较，数组就可比较：

```go
fmt.Println([2]string{"a", "b"} == [2]string{"a", "b"}) // true
```

数组元素不可比较，数组就不可比较：

```go
// _ = [1][]int{} == [1][]int{} // 编译错误
```

结构体要求所有字段都可比较：

```go
type Key struct {
	TenantID int64
	UserID   int64
}

fmt.Println(Key{1, 2} == Key{1, 2}) // true
```

新增不可比较字段会改变整个结构体的可比较性：

```go
type Key struct {
	TenantID int64
	UserID   int64
	Tags     []string
}

// Key{} == Key{} // 编译错误
```

公共 key 类型要谨慎加字段。

## 3. 为什么 map key 必须可比较？

map 查找需要哈希和相等判断。

```go
type Key struct {
	Path string
	Lang string
}

m := map[Key]string{}
m[Key{Path: "/home", Lang: "zh"}] = "首页"
```

`Key` 可比较，所以 map 能判断两次查询是不是同一个 key。

slice 不可作为 key：

```go
// m := map[[]byte]int{} // 编译错误
```

如果你想用 `[]byte` 的内容做 key，可以转成 string：

```go
func Lookup(m map[string]int, b []byte) int {
	return m[string(b)]
}
```

但这里通常会发生转换和分配，热点路径要用 benchmark 验证成本。

## 4. interface 比较为什么可能运行时 panic？

接口的静态类型可以比较，但比较时要看动态值是否可比较。

```go
var a any = "go"
var b any = "go"
fmt.Println(a == b) // true
```

动态值是 slice 时：

```go
var a any = []int{1}
var b any = []int{1}

fmt.Println(a == b) // panic
```

同理，`map[any]V` 也可能 panic：

```go
m := map[any]int{}
m["ok"] = 1

// m[[]int{1}] = 2 // panic
```

如果 key 来自外部输入，最好设计明确的 key 类型，不要用 `any` 兜底。

## 5. 指针相等和内容相等有什么区别？

指针相等比较的是地址是否相同。

```go
a := User{Name: "Tom"}
b := User{Name: "Tom"}
c := &a

fmt.Println(&a == &b) // false
fmt.Println(&a == c)  // true
```

如果业务要比较内容，需要比较字段：

```go
func EqualUser(a, b *User) bool {
	if a == nil || b == nil {
		return a == b
	}
	return a.Name == b.Name
}
```

不要因为指针可比较，就默认它满足业务相等语义。

## 6. 泛型里的 `comparable` 解决了什么问题，又不解决什么问题？

它让编译器知道类型参数支持 `==` 和 `!=`。

```go
func Contains[T comparable](items []T, target T) bool {
	for _, item := range items {
		if item == target {
			return true
		}
	}
	return false
}
```

如果写成 `T any`，函数体里不能用 `==` 比较 `T`。

但 `comparable` 不保证业务相等语义正确：

```go
type User struct {
	ID int64
}

// 比较 User 会比较所有字段，不一定等于“同一个用户”的业务定义。
```

如果业务相等只看 ID，就应该写明确的比较函数，而不是机械依赖 `==`。
