# 024. 泛型 - 面试追问

## 1. 泛型和 `any` 最大区别是什么？

泛型保留静态类型信息，`any` 会把类型检查推迟到运行时。

泛型：

```go
func First[T any](items []T) (T, bool) {
	if len(items) == 0 {
		var zero T
		return zero, false
	}
	return items[0], true
}

n, _ := First([]int{1, 2})
```

`n` 的类型是 `int`。

`any`：

```go
func FirstAny(items []any) (any, bool) {
	if len(items) == 0 {
		return nil, false
	}
	return items[0], true
}

v, _ := FirstAny([]any{1, 2})
n := v.(int)
```

需要断言，断言错了会 panic。

## 2. 约束里的 `comparable` 有什么作用？

它允许泛型代码使用 `==`、`!=`，也允许类型作为 map key。

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

如果用 `T any`，上面的 `item == target` 不能编译。

但 `comparable` 不代表业务相等一定正确。指针可比较，但比较的是地址；结构体可比较，但比较的是所有字段。

## 3. `~int` 和 `int` 作为约束有什么区别？

`int` 只允许类型本身是 `int`。

```go
type OnlyInt interface {
	int
}
```

`~int` 允许底层类型是 `int` 的新类型。

```go
type IntLike interface {
	~int
}

type UserID int

func Add[T IntLike](a, b T) T {
	return a + b
}

Add(UserID(1), UserID(2)) // 可以
```

公共库里如果希望支持调用方自定义的新类型，通常需要考虑 `~`。

## 4. 泛型什么时候比接口更合适？

当核心是“同一算法处理不同类型”，而不是“依赖一组行为”时，泛型更合适。

```go
func Reverse[T any](items []T) {
	for i, j := 0, len(items)-1; i < j; i, j = i+1, j-1 {
		items[i], items[j] = items[j], items[i]
	}
}
```

接口更适合行为抽象：

```go
type Sender interface {
	Send(context.Context, Message) error
}
```

如果你只是需要调用 `Send`，接口比泛型更直接。

## 5. 什么时候不应该为了减少重复使用泛型？

当重复很少，或者抽象会让业务语义变模糊时。

```go
func ValidateEmail(s string) error
func ValidatePhone(s string) error
```

这两个函数都接收 string，但校验规则不同。强行抽泛型没有价值。

还有一种情况是约束很复杂：

```go
func Do[T interface{ ~string | ~[]byte; SomeMethod() }](v T) {}
```

如果读者理解约束比理解业务还难，就应该重新评估。

## 6. 公共泛型 API 的约束为什么要尽量小？

约束越大，调用方能传入的类型越少。

不必要的窄约束：

```go
func Keys[M ~map[string]int](m M) []string
```

它只能处理 `map[string]int` 这一类。

更通用：

```go
func Keys[M ~map[K]V, K comparable, V any](m M) []K {
	keys := make([]K, 0, len(m))
	for k := range m {
		keys = append(keys, k)
	}
	return keys
}
```

公共 API 要让约束准确表达“函数真正需要什么”，不要把实现习惯变成调用方限制。
